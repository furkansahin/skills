---
name: jeremy-lens
description: The ubicloud/Clover code-quality standard (the "Jeremy Evans review lens"), made the standing bar for all our code. Apply when writing OR reviewing Ruby (Sequel/Roda/Rodauth) in the ubicloud repo -- specs, models, progs, migrations, lib -- or Go in terraform-provider-ubicloud, and before opening, committing to, or self-reviewing any PR in those repos. Covers the mocking boundary, spec discipline, rescue/transaction scope, provider dispatch, model organization, migrations, comments, commit/PR assembly, and the Go translation.
---

# The Jeremy lens

This is the review standard Jeremy Evans (Sequel/Roda author, Clover
maintainer) applies, distilled from ~61 comments on PR #5262 plus later
corrections. It is the baseline, not advice. Every spec written, every
prog/model change landed, every PR assembled conforms -- Ruby in ubicloud
and Go in terraform-provider-ubicloud alike.

**The meta-rule.** If a problem looks like it needs one of the prohibited
shapes (a model-level stub, `allow_any_instance_of`, a broad `rescue`, a
mocked Sequel association, an expect-count-only spec, a defensive `&.foo`),
the default move is **stop and find the right shape**, not "just this once."
Each rule below is a review round-trip already paid for once; re-paying it is
the cost of ignoring this.

**But verify before applying.** Jeremy is mostly right, not infallible (a
`map!`-on-`group_by` suggestion on #5262 was invalid; specs caught it). Read
his intent (he sometimes writes "ensure" for "expect"). "Up to you" means
reply-only: acknowledge the trade, do not force the change. Defer suggestions
blocked on another unlanded PR and note the blocker.

---

## 1. The mocking boundary -- the single most important rule

Mock ONLY at the network boundary. Everything below it runs for real.

| Thing | Stub it? |
|---|---|
| Sequel models / DB rows | **No** -- use the real row |
| Sequel associations | **No** -- use the real association |
| Memoized accessors on models | **No** -- let them run; stub their dependency |
| SDK client constructors (`Aws::EC2::Client.new`, `Google::Cloud::*::Rest::Client.new`) | **Yes** -- this is the wire |
| HTTP libraries / raw sockets (Excon connection) | **Yes** |
| `Time.now` / `SecureRandom` | Yes, sparingly |
| Anything else | Stop and ask before stubbing it |

Mocking a dataset method to return a Set/Array is "almost always a terrible
idea" (verbatim). `allow_any_instance_of` is a code smell reserved for when
you truly cannot get the real instance. Don't mock the thing you can check:
for a semaphore incrementer, attach a real strand and assert
`Semaphore.where(...).count`, not `expect(model).to receive(:incr_xyz)`.

Incident (2026-05-12): an agent was about to swap `allow_any_instance_of(
LocationCredentialGcp)` for `allow(credential).to receive_messages(...)` --
same model-level-stub shape, narrower scope. Both wrong. The right answer was
to stub the SDK constructor (`allow(Google::Cloud::Compute::V1::*::Rest::
Client).to receive(:new)`) and let `vm.location`,
`location.location_credential_gcp`, and the client's own memoization all run
for real. The canonical pattern already lived in
`spec/prog/vnet/gcp/vpc_nexus_spec.rb`.

---

## 2. Specs

**Real over mock -- DB objects.** Never
`allow(location).to receive(:location_credential_gcp)`. Materialize
`LocationCredentialGcp.create_with_id(location, ...)` and let the association
resolve. Same for `vm.ip4_string`: use `AssignedVmAddress.create(...)` (helper
`add_ipv4_to_vm` at `spec_helper.rb:334`), not a stub.

**Real over mock -- FirewallRule / port ranges.** `instance_double(
FirewallRule, port_range: 22..23)` bypasses Postgres `int4range` half-open
semantics. Use `FirewallRule.create(firewall_id:, cidr:, port_range:
Sequel.pg_range(22...23), protocol: "tcp")`.

**Real strand, correct id.**
`let(:st) { Strand.create_with_id(model, prog: "...", label: "start") }` so
`subject_is :model` resolves `nx.model` via `strand.id`. Never `Strand.create`
without `create_with_id`; never stub `nx.model`.

**Two-frame stack when production `push`es.** A pushed sub-prog needs
`[child_frame, root_frame]` with a `link`:

```ruby
let(:st) {
  child = {"subject_id" => vm.id, "link" => ["Vm::Gcp::Nexus", "wait_sshable"]}
  vm.strand.update(
    prog: "Vnet::Gcp::UpdateFirewallRules", label: "update_firewall_rules",
    stack: Sequel.pg_jsonb_wrap([child] + vm.strand.stack)
  )
}
```

Single-frame setups mask child-vs-parent frame bugs.

**`expect` over `allow`, with a count.** `allow(X).to receive(:Y) { |a|
captured << a }` passes even if Y never fires. Use
`expect(X).to receive(:Y).exactly(N).times` when the count is derivable. Keep
`allow` only when the count is genuinely variable (retry path) and comment the
tolerance. Converting `allow` -> `expect` in a sweep is high-leverage: it
exposes "this stub never fired" latents across a file.

**Test args, not just call count.** Don't stop at `.twice`; if prod passes
specific args, assert them:

```ruby
bound = []
expect(crm).to receive(:create_tag_binding).twice do |b|
  bound << b.tag_value_namespaced_name; op_double
end
expect(bound).to contain_exactly(fw_tag_value, subnet_tag_value)
```

**`.and_call_original` on every `Clog.emit` assertion.** Never stub emit out;
assert args AND let it run:
`expect(Clog).to receive(:emit).with("msg", anything).and_call_original`.

**`refresh_frame`, never direct stack mutation.** FORBIDDEN:
`st.stack.first["k"] = v; st.modified!(:stack); st.save_changes;
nx.instance_variable_set(:@frame, nil)`. Use
`refresh_frame(nx, new_values: {"k" => v})` (child frame) or
`refresh_frame(nx, parent_values: {...})`. If the helper lacks a case you
need, extend it -- do not reach for direct mutation. `instance_variable_set(
:@frame, nil)` outside `refresh_frame` is itself the smell.

**Instance-method seams, not `DB[:table]` stubs.** If prod does
`DB[:private_subnet].where(...).select_map(...)`, extract a dataset-returning
method (`used_firewall_priorities_ds`) and stub that, or better, insert real
rows so the real query runs.

**`not_to raise_error` -- standalone only.** Keep it when it is the ONLY
assertion (a no-op test); strip it when other `expect`s exist. Never wrap
`expect { x }.not_to raise_error` when a raise would already fail with a
backtrace.

**Test plumbing on `Prog::Test::Base`, not production models.** Shared
teardown (`verify_timelines_destroyed`) goes on `Prog::Test::Base`. Never add
`destroy_remaining_timelines` to `PostgresTimeline` just because two test
progs share it.

### `let` discipline

- **Avoid `let!`.** Use `let(:foo) { ... }` + a sibling `before { foo }` so
  the eager evaluation is visible, not hidden in syntax.
- **DRY across lets.** If `let(:nic) { vm.nics.first }` exists, write
  `let(:ps) { nic.private_subnet }`, not `vm.nics.first.private_subnet`.
- **Sequel mutators return the model** -- drop the redundant trailing line:
  `let(:st) { gcp_vpc.strand.update(...) }`, no `gcp_vpc.strand` after. Same
  for `add_X`/`remove_X`.
- **Hoist long SDK namespaces** to a file-scope local (not a constant):
  `v1 = Google::Cloud::Compute::V1` above `subject`, then `v1::Instance.new`.

---

## 3. Production code

**Bulk dataset ops over per-row loops.** N queries where one suffices is a
smell. `Semaphore.incr(remaining.map(&:id), "destroy")` (accepts id or array,
one CTE INSERT) over `each(&:incr_destroy)`.
`servers_dataset.distinct.select_map(:timeline_id)`, not load-then-map.

**Accept `Sequel::Model` instances in model-API surface.**
`create_with_id(model)`, not `create_with_id(model.id)` -- a `.id` typo
silently passes `nil.id`; the instance form fails loudly. Applies to
`Sshable.create_with_id(vm)`, `NicGcpResource.create_with_id(nic, ...)`, etc.

**Narrow rescue, explicit `nil` on swallow.** Broad `rescue => e` in
destroy/cleanup hides bugs. Catch specific types, re-raise on non-matching
status:

```ruby
rescue Google::Apis::ClientError => e
  raise unless e.status_code == 409
  ...
rescue Google::Cloud::NotFoundError
  # Already released
  nil
end
```

An empty rescue body reads as "forgotten," not "intentional."

**DRY via extraction -- private helpers first, modules second.** When the
rescues repeat too, the extracted helper OWNS them; don't leave rescue clauses
in callers (`ensure_crm_resource` folded two 39-line bodies to 17 and 12).

**Return-type consistency across all paths.** If `assemble` returns a
`Strand` on the happy path, the rescued path returns a `Strand` too (call
`.strand` on the looked-up model, not the model).

**Named LRO slots + documented invariants.** `save_gcp_op(op_name, scope,
scope_value = nil, name: "gcp_op")` stores `{name}_name/_scope/_scope_value`
in the frame; distinct `name:` slots for concurrent ops. Codify the invariant
as a comment ("Labels that call poll_gcp_op must only be entered after a
save_gcp_op(...); hop_wait sequence").

**Frame-layout discipline.** Pending-op state -> child frame (`frame[...]` =
`strand.stack[0]`). Cross-label/root state (`gcp_zone_suffix`, initial params)
-> root frame (`strand.stack[-1]`), documented where read.

**Shell-injection safety via placeholders.** `NetSsh.command(":public_keys",
public_keys:)` shell-escapes safely. Never base64 round-trip to smuggle user
data through a shell; drop `require "base64"` when you remove it.

**Two-phase CHECK constraints on large tables.** (1) `ADD CONSTRAINT ...
NOT VALID` (fast, new rows only). (2) separate `Sequel.migration {
no_transaction; change do run "ALTER TABLE x VALIDATE CONSTRAINT y" end }`.
One `alter_table` block per migration; never multiple DDL in `no_transaction`.

**Architecture decisions documented in source, not just the PR.** If a
reviewer asks "why this priority band / tag scheme / naming?", the answer gets
committed next to the code (or in `doc/`).

---

## 4. Rescue scope and transactions -- critical

- **Never rescue `Sequel::*` inside a `DB.transaction`.** A raised Postgres
  transaction is poisoned: every later Sequel call in scope re-raises
  `Sequel::DatabaseError`. Use a savepoint (`DB.transaction(savepoint:
  true)`), or let it bubble and retry at the calling label.
- **Every strand label body IS a `DB.transaction`** (respirate wraps it). So
  the rule applies unconditionally inside any label, even without a literal
  `DB.transaction do`. Idempotence in a label CANNOT be `begin; create_with_id;
  rescue Sequel::UniqueConstraintViolation; end` -- the rescue poisons the
  rest of the label. Instead: piggyback on an existing frame-key guard
  (`unless strand.stack.first.key?("foo")`), do a fresh PK lookup
  (`Model[resource.id]`), or move the create to `Nexus.assemble` (outside the
  label transaction).
- **Rescuing SDK/HTTP errors in a transaction is fine** when no Sequel state
  mutated before the exception (e.g. `rescue Google::Cloud::AlreadyExistsError`
  right after an `insert`). Those SDK errors don't poison the Sequel txn.

---

## 5. Model organization and provider dispatch

- Provider-specific behavior lives in `model/<provider>/*.rb`, wired via
  `plugin ProviderDispatcher, __FILE__`. No `if location.gcp?` branches in
  shared models.
- Every provider implements the full dispatched method set. A no-op
  (`def metal_foo(x); end`) beats a missing method -- class load raises
  "Not all methods implemented by all providers" otherwise.
- **Sequel associations go in the base `model/<name>.rb`**, including
  associations to provider-specific sub-resources (`vm_gcp_resource`,
  `aws_instance`, `nic_gcp_resource`). They sit next to each other with the
  rest of the model's associations. `model/<provider>/<name>.rb` reopens the
  class ONLY to open `module <Provider>` and define the dispatched
  `<provider>_<meth>` private methods -- nothing else. Grep `model/gcp|aws|
  metal/*.rb` for top-level `one_to_*`/`many_to_*` outside the provider module;
  any hit is a misplacement.

---

## 6. Migrations and schema

- `change do ... end` for reversible ops (Sequel auto-generates the inverse);
  top-level `revert do` when the up-migration is the inverse of a block.
- `collate: '"C"'` on ASCII-only text columns (names, ids, path fragments) --
  match `location_credential_gcp`, `nic_gcp_resource`.
- **Fold un-shipped rename/alter migrations into their original.** If a table
  and its rename are both unreleased, they are one migration.
- Drop `null: false` from primary-key foreign_keys (PKs are not-null by
  definition).
- **Migrations, schema changes, and the `cache/schema.cache` update go in the
  branch's FIRST commit, always.** When work partway through a branch needs a
  migration, rebase to fold the migration + cache hunks into commit 1 (history
  rewrite for this is expected, even on a pushed draft; verify tree-identity
  with an empty `git diff old-tip new-tip`). Reviewers and per-commit tooling
  then see the final schema up front.

---

## 7. Data-model idioms and allocation hygiene

- `Sequel.pg_range(a...b)` for port ranges (half-open).
- `.b` suffix for ASCII-only string literals, not
  `.dup.force_encoding("ASCII-8BIT")`.
- `Time.now` over `DateTime.now`. `to_set(&:x)` over `map(&:x).to_set`.
  `Set.new(enum, &block)` over accumulate-into-`.to_set`.
- `|| [].freeze` over `|| []`; `x&.method || [].freeze` over `(x ||
  []).method` (skips the intermediate array when upstream is nil).
- `.select_map(:id)` over `.all.map(&:id)` (skips row instantiation).
- **Don't `.freeze` a string with no mutation to prevent.** A value consumed
  only by `puts`/`$stdout.write`/`IO#write` is never mutated by them, so the
  `.freeze` buys nothing -- drop it (PR #5928: Jeremy cut `(JSON.generate(out)
  << "\n").freeze` to `JSON.generate(out)`). Freeze is for shared constants and
  frozen-literal-magic-comment gaps, not for one-shot locals.
- **Keep the line-framing at the write boundary, not in the serializer.** A
  `generate`/`serialize` helper returns the bare payload; the writer appends the
  `"\n"` (`raw = generate_line(out) << "\n"`). Baking the newline (and its
  allocation) into the generator couples two concerns and forces the freeze
  above.
- **Byte caps use `bytesize`/`byteslice`, not `length`/`[]`.** A size cap on
  data that may not be valid UTF-8 (SSH stderr/stdout, command output, log
  volume) is a byte budget, so measure and slice in bytes: `(s.bytesize >
  CAP) ? s.byteslice(-CAP..) : s` (PR #5925). `String#length`/`[]` count
  characters, which miscount on invalid-encoding bytes and character-slice
  where you meant to byte-slice. Assert the cap with `.bytesize` in specs, and
  add a multibyte case (`"é" * n`) so a revert to `.length` fails loudly.
- `Strand.create_with_id(model, ...)` over `Strand.create { it.id = model.id }`.
- `Model.generate_uuid` in specs over `SecureRandom.uuid`. `.new_with_id` over
  `.new.tap { it.id = X }`.
- `Enumerable#find { ... }` over manual each-with-break-and-assign.

---

## 8. API / codepath patterns

- **Insert-first + rescue `AlreadyExists`** over get-insert-get (one fewer
  network call on the happy path; the rescue runs only on the contested path).
- **Make kwargs required if all callers already pass them** -- drop the
  default, force the contract at every call site.
- **Nested-hash frame layout** (`{foo: {name, scope, value}}`) over flat-key
  prefixing (`{foo_name, foo_scope, foo_value}`): related fields travel
  together; partial updates can't leave inconsistent state.

---

## 9. Comments -- terse, timeless, ASCII

- **Default to no comment.** Rationale/history goes in the commit message
  (Clover follows the Linux-kernel convention). A 9-line lock-protocol essay or
  a "why we drop this" paragraph gets cut to a one-liner or deleted; the reason
  moves to the commit. ("Ruby is already a text-like programming language.")
- **A comment states a constraint the code can't show** -- never where it came
  from, what the next line does, or that a change is correct.
- **No non-ASCII in code or comments.** `*` not the multiply sign; `--` or `-`,
  never em/en-dashes.
- **Misleading code is worse than missing code.** Do not ship a `decr_*` /
  `clear_*` / `finalize` / `drain` / `acknowledge` call whose semantics assert
  a precondition the surrounding code has NOT established -- even when
  "harmless today," "needed once we add the consumer," or "mirrors the sibling
  path." Such a line is a documented assertion that lies.
- **Drop defensive noise:** `&.destroy` / `&.cleanup` on objects nothing
  upstream creates (masks regressions); `instance_variable_set(:@foo, nil)` on
  an ivar never set; single-caller class-method wrappers (`def
  self.lock_subnet!(ps); ps.lock!; end` with one caller is noise).

---

## 10. Commits and PR assembly

- **Subject:** imperative, capitalized, aim ~50 chars. **No prefix ever** --
  not `postgres:`, not `(feat)`. Lead with the area in plain words ("AWS
  Postgres open 443 to the GuardDuty endpoint"). A colon inside a literal token
  (`guardduty:SendSecurityTelemetry`) is not a prefix and is fine.
- **Body:** one tight paragraph (2-4 lines) of why + what, **hard-wrapped at
  72**. Not multi-paragraph. The "how" is the diff.
- **EXCEPTION -- subtle mechanisms (the fdr rule, PR #5950 rewriting #5932):**
  a commit touching a subtle mechanism (concurrency, invariants, scheduling)
  earns as many paragraphs as comprehension needs, and MUST NOT be a
  compression of the investigation trace. Open by NAMING the canonical
  concept ("compare-and-set with strand.schedule as its optimistic
  concurrency control key"); state the invariant and where it lives; argue
  the property holds by construction (and when it is stronger than what it
  replaces -- "previously coincidental, now deterministic"); name the cost.
  History is evidence for the design forces, never the frame. Subject states
  the achieved property ("Make wake writes non-demoting yet always
  detectable"), not the implementation ("Use LEAST minus a microsecond").
  Comments split the same way: the invariant-carrying declaration gets the
  full explanation; use sites name the mechanism and cross-reference it.
  Litmus test: could someone who was not in the investigation reconstruct
  the design, alternatives, and costs from the message alone?
- **No `Co-Authored-By: Claude` trailer** on ubicloud commits (override the
  global default).
- **Migration/schema/cache -> first commit** (see section 6).
- **Per-commit coverage:** each commit is coverage-clean in isolation
  (`bundle exec rake coverage_pspec` exit 0 = 100% line+branch; trust the exit
  code, not the racy printed %). CI also runs `rake frozen_pspec`
  (`CLOVER_FREEZE=true`) -- never mutate frozen global state in specs
  (`DB.loggers <<` raises FrozenError there even when local cberp is green).
- **Codex adversarial review on the closing commit** is acceptance criteria;
  verify the `-o` file is non-empty AND newer than the run before trusting the
  verdict.
- **Run rubocop before every push.**

---

## 11. Correctness discipline

- **Reachability must have an answer.** "Can this actually happen?" is
  answered in code or the commit message before a guard/branch ships.
- **No partial fix on a falsified premise.** When a codex/review finding
  contradicts an explicit premise of the task: stop, do not ship the part that
  depends on it, reduce scope, file a follow-up naming the falsified premise
  and the empirical check needed, and note the trade. "Currently dormant" is
  not a correctness rationale once the premise is disproved. The issue body is
  no longer a trustworthy spec.
- **Verify claims against actual code.** AI (including yours) confabulates and
  agrees too readily. Grep the real code before asserting a mechanism, a flag
  name, or a default. This applies to your own summaries too.

---

## 12. The Go translation (terraform-provider-ubicloud)

The lens carries to Go verbatim in spirit:

- **Guard reachability** -- same as section 11.
- **Terse, timeless, ASCII comments** with no stack traces or process
  references.
- **Fakes at the wire boundary only**, matching the OpenAPI shapes -- never
  fake internal layers.
- **No single-caller indirection.**
- **Explicit contracts over silent defaults** -- the required-kwargs principle.
- **Narrow error classification** -- the narrow-rescue principle.
- **Commit bodies hard-wrapped at 72.**
- **Verify-before-applying** for review/Codex findings.

---

## Quick pre-PR checklist

1. Any stub not at the network boundary? -> replace with the real object.
2. Any `allow` that should be `expect` with a count and asserted args?
3. Any `rescue Sequel::*` inside a label or transaction?
4. Any `st.stack[...] =` or `instance_variable_set(:@frame, nil)`?
5. Any association declared in a `model/<provider>/` file?
6. Migration + `cache/schema.cache` in the first commit?
7. Comments: any that belong in the commit message instead? Any non-ASCII?
8. Any defensive `&.foo` / cleanup call asserting an unestablished precondition?
9. Commit subjects prefix-free, bodies wrapped at 72, no Claude trailer?
10. Per-commit `coverage_pspec` exit 0, `frozen_pspec` clean, rubocop clean?
