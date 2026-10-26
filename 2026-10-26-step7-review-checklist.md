# AI-OS #7 review checklist (Claude, 2026-10-26)

Use this to check Codex's work on #7. Paste Codex's final report plus the command output below, and Claude marks each line PASS / FAIL.
Source: STATUS.md NEXT #1 (Codex-review conditions, approved 2026-10-24) + Parking Lot.

## Before anything moves (gate)
- [ ] **Disk:** free space checked before each copy. Mini was 95% full (~12 GB). `df -h ~`
- [ ] **Map exists:** one table that lists every Codex project ID, root, chat, worktree and unique file → its destination. It's written BEFORE any move.
- [ ] Every Drive/ChatGPT repo from the inventory (58 listed / 45 markers) is in the map, or marked "skip" with a reason.
- [ ] **Bundle coverage:** each repo being moved is in `~/Workspace/_archive/2026-10-repo-bundles/MANIFEST.md`, or its uncommitted/untracked files are backed up separately.
- [ ] Nothing under the ⛔ #8/#19 blocker (TravOS, MIKE, Hermes recovery repos) is moved or touched.
- [ ] Secret files (`.env*`, `.secrets/`, `settings.local.json`) are not copied anywhere new.

## Moves (§B)
- [ ] Originals preserved: copied, not moved. No deletes (deletes are #19).
- [ ] Each new home is under `~/Workspace/Code/<name>`. `git status` and `git log -1` there match the original.
- [ ] Repos with a GitHub remote: `git fetch` works from the new home.
- [ ] `agent-config/repos.txt` is updated for any new repo with a remote.

## Codex projects repointed (§L)
- [ ] `~/.codex/config.toml` was backed up first (to `~/Workspace/_archive/...`).
- [ ] `[projects."/Users/trav"]` (whole home trusted) is removed.
- [ ] Old `Documents/ChatGPT` and scratch `[projects.*]` entries are pruned or repointed.
- [ ] `~/Workspace` is still trusted. `setup-machine.sh` (check mode) reports `codex project ~/Workspace trusted`.
- [ ] **Fresh chat** in each repointed project opens with cwd = the new path.
- [ ] **Resumed chat** (an old one) also opens in the new path. Test at least 2.
- [ ] Both misrouted chats in `Code/VoiceCapture` are moved to the right project:
  - [ ] "Plan land listing analysis"
  - [ ] "Locate Fusion records and agreement"

## Done-test + paperwork
- [ ] `setup-machine.sh` (check mode) on the mini: 0 failures.
- [ ] STATUS.md: #7 moved to DONE with date and commit hashes; the NOW line updated to the next step.
- [ ] DECISIONS.md: map location and any "skip" calls recorded.
- [ ] Parking Lot line "Codex still trusts the whole home folder…" struck through.
- [ ] Only one writer touched the vault during #7 (no Drive "conflict" copies): `find ~/Workspace/TravisVault -name "*conflict*" -newer STATUS.md`

## Commands to paste back
```bash
df -h ~
grep -n '^\[projects' ~/.codex/config.toml
ls ~/Workspace/Code
bash ~/Workspace/agent-config/scripts/setup-machine.sh | tail -15
```
