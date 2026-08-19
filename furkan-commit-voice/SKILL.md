---
name: furkan-commit-voice
description: Furkan Sahin's personal commit-message voice for the ubicloud/ubicloud repo -- phrasing, structure, and recurring habits layered on top of COMMIT_MESSAGES.md's mechanical rules (imperative mood, 72-col wrap, no prefixes -- this skill does not repeat those). Apply when drafting or rewriting a commit message on Furkan's behalf: a git commit body, a PR's squash-commit message, or a commit-message rewrite requested during review.
---

# Furkan's commit voice

COMMIT_MESSAGES.md sets the mechanical shape: imperative subject, ~50-char
subject / 72-col body wrap, no conventional-commit prefixes, body explains
why/what not how. This skill does not repeat any of that -- it's what he does
*inside* that shape, drawn from his actual commit history.

## Subject line: imperative is dominant, but not a hard rule for him

The style guide says imperative ("Add", "Fix"). He mostly follows it, but a
descriptive third-person-singular subject ("Adds", "Fixes", "Removes",
"Prepares", "Blocks", "Makes") is a real minority, most common in
2023-2024 commits and still turning up occasionally since:

- "Skips VMs at refresh_mesh if the ephemeral_net6 is nil"
- "Fixes when /failover is empty for hetzner api"
- "Prepares virtual networking for UI changes"
- "Blocks IP spoofing via nftables"

Default to imperative -- it's the majority form throughout, and increasingly
so in 2025-2026 commits -- but don't treat the -s form as an error to stamp
out if it's what falls out naturally.

Occasionally the subject is a full clause, or two clauses joined by a comma,
narrating the change as a small fact rather than commanding an action:
"guest_mac is removed, we use mac from controlplane"; "Fix DHCPv4 Lease Time
(Really set it to 6h)". Rare, but it shows the subject-line grammar isn't
treated as absolute.

Migration-only commits get no standardized suffix casing -- "Migration",
"migration", "mig", "-migration-", "Mig 1"/"Mig 2" all appear, sometimes
within the same feature's commit sequence, and these are often a bare noun
phrase with no body at all ("Add MinIO migration file"). Don't invent a
"correct" casing he never used -- just flag the commit as a migration however
reads naturally.

## Bug fixes: usually symptom first, cause second, fix last

This is the most consistent structural habit in the corpus, and the default
to reach for. The shape:

1. What broke -- often a pasted, verbatim error/log/stack-trace excerpt, not
 a paraphrase.
2. "This is happening because..." / "The reason is..." / "The problem is..."
3. What the fix does.

> We just skip the vms if they don't have ephemeral_net6 assigned yet.
> Because they don't get it at the starting step... the refresh_mesh step
> gives this error;
> `SshError: .../netaddr-2.0.6/lib/util.rb:224:in 'parse_IPv4'...`
> This is happening because the dst_ip is nil

> check_pulse turns every error into a "down" reading. The checkup then
> calls available?, which asks the server over SSH as postgres and
> succeeds, so no page occurs. The failure stops last_known_lsn instead.

When you have a real error message, log line, or stack trace to include,
include it verbatim -- don't summarize it away. This recurs from the first
commit in the corpus through the most recent.

The exception: when the bug's actual cause is a specific prior change --
a regression from an earlier commit or PR -- he'll sometimes lead with that
causal context instead, since "symptom" and "cause" collapse into one fact:

> With a recent change, we moved the tcp 22 rule into an internal firewall
> which are attached to VMs once they are up and running. However, AWS
> security groups are created through VPCs which are part of
> private_subnets. Therefore, not having the tcp 22 rule on the
> private_subnet firewall breaks the initial VM provisioning in AWS.
> (commit 00e18c20, "Fix Ubi PG on AWS VPC creation with tcp 22 allow rule")

> We broke this with the commit ec3f786. Only old clusters are impacted.
> (commit 15b8427a, "Fix Nic provisioning for old AWS subnets")

Reach for symptom-first as the default; drop straight into cause only when
the cause is itself a named prior commit or PR the reader needs as context
before anything else makes sense.

## Cite the specific prior commit or PR a change follows from

Beyond bug regressions, he routinely grounds a commit's rationale by citing
an earlier PR (by URL) or commit (by SHA) directly in the body -- not to
credit whoever reviewed *this* change, but to trace the lineage of the code
itself:

> With the PR https://github.com/ubicloud/ubicloud/pull/2272, we have
> introduced source port overwriting...

> This fixes commit bb3a0ec68544207e2d178bd85b79fc6b5d940b7e.

> In the very first commit 4a00e905, we have introduced a change...

> Basically implements the PR https://github.com/ubicloud/ubicloud/pull/4310

Use this whenever a commit reworks, fixes a regression from, or extends
earlier work -- it's a separate habit from crediting a reviewer's suggestion
(see the Recent register section below), and both can appear in the same
body.

## Causal connectors do real work

He makes the logic of a change explicit rather than implicit, constantly,
with two tools: sentence-initial "Therefore," (66 occurrences, spread evenly
across the corpus) and mid-sentence ", so" (103 occurrences, more dominant in
the denser recent register). Use either to chain an explanation to its
consequence instead of leaving the reader to infer it:

> Therefore, we need to add another tool to handle router advertisement.

> Therefore, we are also fixing a bug that would cause a crash in case the
> host prefix is too long...

## "We" narrates the system; "I" narrates a judgment call

Default voice is "we" -- what the code/system does. He switches to "I"
specifically for a personal judgment call, an exploratory benchmark he ran
himself, or a trade-off he's owning -- not as a random stylistic tic:

> I have also performed a very simple, non-scientific benchmark to see the
> impact of this change. I have created a postgres VM on my development
> host... I ran this command directly from the host against the postgres VM
> to eliminate any remote connection related latency from the picture.

> I chose latter considering it's much cheaper for engineering.

If the sentence is describing what the code does, stay in "we". If it's
describing a decision he made or an investigation he personally ran, switch
to "I".

## Ad hoc numbered lists, semicolon-led

For multi-part reasoning or itemized changes, especially useful for older /
denser-2023-2024-style commits: end the lead sentence with a bare semicolon,
then number the parts.

> There are 2 main themes;
> 1. For a given subnet, redirect the whole traffic...
> 2. We are assigned /64 subnet, we split it to /79 chunks...

This largely gave way to plain prose or dash bullets by 2026 -- still a
valid tool for a genuinely multi-part rationale, just not the only one.

## "Looks like ..." -- reserved for other people's systems

A mild hedge introducing something learned by reading logs/docs/source of a
third-party system (kernel, gem, hosting-provider API) -- never used to
describe his own code's behavior:

> Looks like the recently published ubuntu-jammy images come with kernel
> 5.15.0-101-generic which has the patches applied

## Narrate *why* a change is split across commits

When a change is staged across multiple sequenced commits, say so
explicitly. Most often this is about zero-downtime/rolling-deploy safety:

> Drop and delete operations in the schema. Mainly separated to its own
> commit to handle the deployment without a respirate restart.

> This commit will be deployed only after the first one reaches production
> and we make sure there is no strand in this label anymore.

But the same habit -- explicitly foreshadowing a later commit -- also covers
plain incremental build-out, with no deploy-safety angle at all:

> This way, we will make use of the filter without passing all the
> vm_hosts in a datacenter in a later commit. (commit 36487e63, "Add data
> center exclusion filter to the allocator")

Either way, the point is the same: don't let a multi-commit sequence's shape
go unexplained. This recurs across the entire timespan of the corpus.

## Recent (2026) register: denser, evidence-led

The chattier "With this commit, we are adding..." / "With this PR, we..."
body-opener (~29 occurrences total) was still a live, unremarkable habit
through nearly all of 2025 -- about a quarter of its uses land in 2025
(Jan-Oct), including as recently as:

> We were only cleaning up the basebackups and forget about the wal files.
> With this commit, we start cleaning up the wal files, too. (commit
> 3c2bd3f0, 2025-07-23, "Fix old backup and wal files delete policy")

But it has zero occurrences in 2026 -- treat it as retired this year, not as
something that faded out back in 2024. The current default is denser:
causally chained multi-clause sentences using colons and dashes to unpack
mechanism, explicit statement of invariants/trade-offs, and concrete
production numbers as evidence instead of adjectives:

> The invariant restored is that a label owning a pending destructive
> operation must be able to abandon it.

> The trade-off: a genuine IAM permission misconfiguration now silently
> leaks the SA instead of blocking destroy forever. This is acceptable
> because leaked SAs are visible in the GCP console, bounded by timeline
> count, and convergence... is strictly better than an infinite retry loop
> that blocks the entire test run.

> Months of wedged and cancelled runs accumulated 991 stale tag keys (of
> the 1000 hard cap) and 29 VPC networks, 26 still holding subnets.

Default to this register for present-day commits; treat "With this commit,
we..." as available but dated, not the primary tool.

Three habits that showed up specifically in a heavily-reviewed 2026 PR
sequence and are worth reusing for similarly substantial, reviewed work:

- A compact test/coverage receipt line at the very end of the body for
 substantial, reviewed changes: "cberp: 6172 examples, 0 failures, 100%
 line / 100% branch. rubocop clean." Don't add this to small or trivial
 commits -- it's a receipt for real review scrutiny, not a habit for every
 commit.
- Crediting a reviewer by name and citing the discussion permalink when a
 commit directly addresses a specific review thread: "Per Jeremy Evans'
 suggestion in PR #5286 review: https://github.com/.../discussion_r..."
- A `Co-Authored-By:` trailer when a commit is genuine joint work with a
 named collaborator: "Co-Authored-By: Philip Dubé <philip.dube@clickhouse.com>"
 (commit b2b42440, 2026-08-07). Use only when the work was actually done
 together, not as a default trailer.

## Small flavor, use sparingly

- Slash-paired compressions for opposite/paired networking values: "private
 ipv4/6 addresses", "VM de/provisionings", "dis/connect 2 subnets". Fits
 networking-adjacent subject matter; don't force it elsewhere.
- A rare, dry, deadpan aside, immediately followed by moving on -- never
 dwelt on: "AWS guys are weird, if you don't pass anything, the bucket is
 created in us-east-1, but the value us-east-1 is not allowed. Such a
 world class API." A handful of these across 732 commits -- don't overuse.

## Avoid

- Exclamation points, even narrating an incident that paged someone -- the
 register stays flat.
- Corporate-safe filler ("in order to better serve", "moving forward",
 "leverage"/"utilize" as generic filler for "use"). When "utilize" does
 appear it's in the earliest 2023 commits and reads old-fashioned, not
 corporate -- don't reach for it as a default synonym for "use".
- Signed-off-by, emoji, marketing enthusiasm ("excited to ship").
- Passive voice that obscures agency ("mistakes were made") -- name "we" or
 "I" as the actor, even for a bug he introduced himself.
- Diplomatic softening of a past decision -- state outright that an earlier
 design "doesn't really work" rather than framing it as an "opportunity for
 improvement".
- Vague scope language ("various improvements", "general cleanup") -- name
 the specific function/entity/column touched, even in a small commit.
- Marketing adjectives ("powerful", "seamless", "robust", "blazing fast") --
 give a measured number instead: "7.5% performance gain", "0.118ms vs
 0.127ms".
