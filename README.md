# skills

Personal Claude Code skills, kept here so they can be pulled onto another
machine instead of rebuilt.

## Skills

- `jeremy-lens/` — ubicloud/Clover Ruby (and terraform-provider-ubicloud Go)
  code-quality standard, distilled from Jeremy Evans' PR review comments.
- `furkan-commit-voice/` — Furkan's personal commit-message voice for
  ubicloud, on top of `COMMIT_MESSAGES.md`'s mechanical rules.
- `furkan-review-voice/` — Furkan's personal PR-review voice (inline
  comments, review verdicts, discussion replies) for ubicloud.

## Install on a new machine

```sh
git clone git@github.com:furkansahin/skills.git
cp -r skills/*/ ~/.claude/skills/
```

Or symlink individual skills instead of copying, to keep them in sync with
this repo:

```sh
ln -s "$(pwd)/skills/jeremy-lens" ~/.claude/skills/jeremy-lens
```
