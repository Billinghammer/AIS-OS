---
project: AI-OS
doc: Design proposal, git-first vault (Drive becomes backup)
status: DRAFT for Travis review (2026-10-26 evening)
author: Claude (cloud session), while Codex runs #7
replaces: design §A (vault on Drive) and the #6 one-way snapshot, if approved
---

# Git-first vault: design proposal

## 1. The problem (why change)
- **Cloud and phone sessions can't see Drive.** Claude on the web, Claude on the phone and Codex cloud only see GitHub. Today a cloud session could not open STATUS.md.
- **Drive makes silent conflict copies** when both Macs write before syncing. Nobody notices until something is lost.
- **The snapshot is slow.** 22 to 27 min a night on the mini, because Drive Stream re-downloads files for git.
- **Drive fills the mini.** `~/My Drive` is 44 GB on a disk that is 95% full.
- **Agents are git-native.** Claude Code and Codex already commit, diff and undo with git. Drive gives them none of that.

## 2. The proposal (one paragraph)
The vault becomes a **private GitHub repo** that is the single source of truth. Each Mac keeps a **local clone** at `~/Workspace/TravisVault` (a real folder, not a link into Drive). A small script syncs it: pull before work, commit and push after. Client data and big files **stay on Drive**. The mini pushes a **nightly one-way backup** of the vault to Drive. Every app reads the same files: both Macs, Codex, Claude Code, Claude on the web and the phone.

## 3. What lives where

| Content | Home | Why |
|---|---|---|
| Notes, STATUS, DECISIONS, AGENTS.md, skills, 00-Context, inbox, audits | **Git (vault repo)** | Text. Every app needs it. History helps. |
| `Transactions/`, `03-People/` | **Drive only** | Client data. Never on GitHub. Already excluded from the snapshot. |
| PDFs, images, video, anything > 10 MB | **Drive only** | Git is bad at big binaries. |
| `.env`, keys, `settings.local.json` | **`agent-config/.env` (local only)** | Never synced anywhere. |
| Code projects (TRAV-OS, MIKE, whisper-flow, ...) | **Own repos in `~/Workspace/Code`** | No repos inside the vault (#6b). |

On each Mac, `Transactions/` and `03-People/` can be **symlinks** from the local vault to their Drive folders. Obsidian and local agents still see them. Git ignores them, so they never reach GitHub or cloud sessions.

## 4. How sync works

**`agent-config/scripts/vault-sync.sh`** (new, same style as `vault-snapshot.sh`):
1. Take the run lock (reuse the snapshot lock pattern).
2. `git pull --rebase --autostash`.
3. If there are local changes: block-list check (`.env*`, `.pem`, `settings.local.json`, `.obsidian/workspace*`), gitleaks on staged files, then commit `vault: <machine> <time>` and push.
4. **On any conflict: stop.** Leave the files as they are, write a note to `00-Inbox/`, exit non-zero. Never auto-resolve.

**When it runs:**
- Every 10 min via launchd on both Macs (part of #11).
- Claude Code + Codex hooks: pull on session start, sync on session end (part of the harness-enforcement handoff).
- By hand: `vault-sync` alias.

**Rules that stay:**
- STATUS.md keeps the one-writer habit (one agent edits it at a time).
- Conflicts are rare with a 10 min cycle and line-based markdown. When they happen, git names the file and the lines, instead of a hidden "conflict" copy.

## 5. Backup to Drive (mini, nightly)
- `rsync -a` the vault working tree to `My Drive/Backups/TravisVault-mirror/`. No `--delete`: Drive version history plus "never delete" is the safety net.
- Weekly `git bundle create` of the full history to `My Drive/Backups/vault-bundles/` (one file, restorable anywhere).
- Result: 3 copies (GitHub, each Mac's clone, Drive) and full history in 2 places.

## 6. Phone
- **Read anything:** the GitHub app, or a Claude session in the Claude app (it sees the repo directly).
- **Edit notes (Android):** Obsidian with the Obsidian Git plugin on a 15 to 30 min pull/push cycle. Or keep phone edits as quick notes into `00-Inbox/` only.
- This meets the "phone apps can read STATUS.md" checkpoint before #12.

## 7. The repo: start clean (recommended)
The current `travis-vault` repo has problems in its history: keys found by gitleaks, plus 7 `Transactions/` and 1 `03-People/` files. Fixing that needs a history rewrite and force-push, which goes against git policy.

**Recommendation:** create a **new private repo** (e.g. `Billinghammer/vault`) from today's clean tree, one first commit. **Archive** `travis-vault` (read-only, kept as history). No force-push needed.
- The leaked keys still need rotating (tonight's reminder). A new repo does not un-leak them.

## 8. Migration plan (after #7, before #11)
Each step has a done-test. Stop on any failure.

| # | Step | Done-test |
|---|---|---|
| M0 | Prereqs: #7 done, #6b nested `.git` folders moved out of the vault, key rotation started | `find TravisVault/ -name .git` = 0 |
| M1 | Freeze: pause the nightly snapshot. One machine (mini) is the only writer for the switch. | No vault writes for the window |
| M2 | Create the new private repo from the current tree. Use the snapshot's `.gitignore` + block list + gitleaks. | gitleaks clean; no `Transactions/`/`03-People/`/`.env` in `git ls-files` |
| M3 | Mini: clone to `~/Workspace/TravisVault.new`. Swap: old link → `_archive`, `.new` → `TravisVault`. Add Drive symlinks for `Transactions/` + `03-People/`. | Obsidian opens it; `setup-machine.sh` check passes |
| M4 | Update `setup-machine.sh`: vault = local clone (not Drive lookup); Codex `writable_roots` → new path; rules links unchanged. | Second `--apply` = 0 changes |
| M5 | Add `vault-sync.sh` + launchd (both Macs) + Drive backup job (mini). Retire `vault-snapshot.sh`. | Edit on MacBook shows on mini within 10 min, and the reverse |
| M6 | MacBook: same as M3/M4. | Fresh `claude -p` and `codex exec` read AGENTS.md from the local clone |
| M7 | Drills: (a) both Macs edit the same line → clean stop + inbox note; (b) mini offline 1 hr then back → sync catches up; (c) restore from the Drive bundle into a scratch folder. | All 3 pass |
| M8 | Drive copy of the old vault stays read-only for 2 weeks, then archive. | Nothing missing reported for 2 weeks |

**Rollback:** until M8, point `~/Workspace/TravisVault` back at the Drive folder. Nothing in Drive is deleted.

## 9. Effect on the build order
- **#6 snapshot:** replaced by `vault-sync.sh` + Drive backup. Simpler, and minutes instead of 25.
- **#11 launchd:** installs vault-sync on both Macs plus the backup on the mini.
- **#13 to #15** ("one host runs jobs, others write inbox notes, check hashes"): git commits replace the hash checks. The one-host rule stays.
- **Drive conflict/offline drill (design §16):** replaced by the git drills in M7.
- **#19 deletions:** the mini can drop `~/My Drive` Mirror mode for Stream, freeing a lot of the 44 GB.

## 10. Risks

| Risk | Guard |
|---|---|
| A secret gets committed | Block list + gitleaks in `vault-sync.sh`, git pre-commit hook (#17), private repo |
| Client data reaches GitHub | `Transactions/`, `03-People/` gitignored + symlinked from Drive; block-list check |
| A sync conflict blocks work | Stops cleanly with an inbox note; fix with one agent prompt |
| A Mac forgets to sync | launchd every 10 min + session hooks |
| GitHub outage | Each Mac has a full local clone; work continues offline, syncs later |
| Big files sneak in | 10 MB size check in `vault-sync.sh` (snapshot already checks 95 MB) |

## 11. Decisions for Travis (tonight)
1. **Go / no-go** on git-first vault, Drive as backup.
2. **New clean repo** (recommended) or keep `travis-vault` history?
3. **Phone:** read-only (GitHub/Claude app) or edit (Obsidian Git on Android)?
4. **`.obsidian/` settings:** keep out of git (as today), or sync them so both Macs look the same?
5. **Sync interval:** 10 min (recommended) or different?
6. **Timing:** after #7 and before #11 (recommended)?

Log the answers in DECISIONS.md, then this becomes design §A v2.
