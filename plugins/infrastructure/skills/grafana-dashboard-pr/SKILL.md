---
name: grafana-dashboard-pr
description: Save a Grafana dashboard into the infrastructure repo via a GitHub pull request — either backing up a UI-built dashboard to files/monitoring/ui-dashboards/ (the normal case) or changing a Terraform-provisioned one under files/monitoring/dashboards/json/. Use this whenever someone wants a dashboard "backed up", "saved to git", "persisted", "committed", "added to the infra repo", or wants a PR opened for a dashboard — including when they only give a dashboard name or a Grafana URL and no git instructions at all, and including when they paste raw dashboard JSON. This skill is meant for non-technical teammates, so use it any time someone hands over a dashboard without spelling out branch names, file paths, or PR mechanics themselves; it can fetch the dashboard from Grafana itself, so they don't need to export anything by hand.
---

# Grafana Dashboard PR

This skill gets a Grafana dashboard into the infrastructure repo as a proper git commit + pull request, so people who don't use git day-to-day can still have their dashboards version-controlled and recoverable.

The person driving this is often non-technical. Keep the conversation in plain language — "I'll open a pull request for you to review on GitHub" rather than talking about refs, HEAD, or index state. Do the git mechanics yourself; don't ask the user to run git commands.

## The two zones — read this before choosing a path

The repo holds dashboards in two places that behave **completely differently**. Picking the wrong one either loses someone's work or silently locks them out of their own dashboard.

| | `files/monitoring/ui-dashboards/` | `files/monitoring/dashboards/json/` |
|---|---|---|
| Purpose | Backup / version history only | Provisioned by Terraform |
| Terraform | Ignores it entirely | Creates a ConfigMap per file |
| Editable in Grafana UI? | **Yes** — stays fully editable | **No** — read-only, Grafana takes it over |
| Source of truth | Grafana | The repo |
| Who it's for | Anyone building dashboards in the browser | Dashboards backing alerts / on-call |

**Default to `ui-dashboards/`.** That is the zone this skill exists for: someone built or changed a dashboard in the browser and wants it safely in git. Copying it there changes nothing about how they work — the dashboard stays theirs and stays editable.

**Never move a UI dashboard into `dashboards/json/` to "make it managed" unless the user explicitly asks for that and understands the consequence:** on the next apply, Grafana marks it provisioned, it becomes **read-only in the browser**, its version history resets, and every future change has to go through a pull request. That is a real loss of workflow for a non-technical user. Say so plainly and get an explicit yes before doing it.

Two facts worth knowing if anyone asks why the split exists:

- Provisioned dashboards are rewritten from the repo whenever their ConfigMap changes **and on every Grafana restart** (the sidecar rewrites the files, so their timestamps are always new). Browser edits to a provisioned dashboard do not reliably survive.
- Dashboards in `ui-dashboards/` have no provisioning file behind them, so nothing overwrites them, ever. They live in Grafana's database, which is SQLite on a single volume — which is exactly why the git backup matters.

## Assumptions this skill relies on

- Claude is running in Claude Code, with a local machine it can run shell commands on.
- The infra repo is `https://github.com/archway-network/infrastructure`.
- The user might give a dashboard URL, a dashboard name, or pasted JSON — all three are fine.

### Grafana endpoints

The same Grafana instance is reachable at two hostnames — same dashboards, same data, either one works:

| Endpoint | Notes |
|---|---|
| `https://grafana.internal.archway.io` | Kong internal ingress. Try this first. |
| `https://grafana.pigeon-saury.ts.net` | Tailscale operator ingress. Fallback. |

Both need Tailscale, and **reads work without a token** on both (anonymous access is enabled, Viewer role), so this skill can fetch dashboards itself. If one endpoint fails, try the other before concluding Grafana is unreachable — one ingress path can be broken while the other is fine, so a failure on both is a much stronger signal that the user is simply off Tailscale.

Pick the working endpoint once at the start and use it for every call in the run. If the user pastes a dashboard URL, just use whichever host their URL is on.

Don't assume `gh` is installed, authenticated, or that the repo is already cloned — check first (Step 0). If anything is missing, **tell the user the exact command to run and wait for them to confirm it's done** — don't install anything, don't run `gh auth login` yourself, and don't clone the repo without them saying go ahead. This is likely someone's first time doing any of this, so keep the explanation plain and point at one command at a time rather than dumping a wall of setup steps.

## Workflow

### 0. Make sure the environment is ready

Run through these checks before touching any dashboard. Skip a check only once you've confirmed it's already satisfied. For each one that's missing, give the user the exact command for their OS and stop there — wait for them to say it's done before moving to the next check.

**Is `gh` installed?**
Run `gh --version`. If it's missing, tell the user to run the install command themselves:
- macOS: `brew install gh`
- Linux (Debian/Ubuntu): `sudo apt install gh` (or point them at https://github.com/cli/cli/blob/trunk/docs/install_linux.md for other distros)
- Windows: `winget install --id GitHub.cli`

**Is `gh` authenticated?**
Run `gh auth status`. If it's not logged in, tell the user to run `gh auth login` themselves and follow the prompts (it opens a browser and asks them to confirm a code) — this is their GitHub identity, so it should always be them completing the login, never Claude. Wait for them to confirm it's done, then re-run `gh auth status` to verify before moving on.

**Is the infra repo cloned locally?**
Check the current working directory, then check for a likely default location (e.g. `~/code`, `~/repos`, `~/dev` if those exist) for a checkout of `archway-network/infrastructure`. If you can't find it, tell the user to run:
```
git clone https://github.com/archway-network/infrastructure.git
```
in whichever parent directory they'd like it (their home directory or wherever they keep code is fine). Once they confirm it's cloned, `cd` into it yourself and continue.

**Is Grafana reachable?** Only needed if you're fetching rather than working from pasted JSON. Try both hosts and keep the one that answers:
```bash
for h in https://grafana.internal.archway.io https://grafana.pigeon-saury.ts.net; do
  printf "%s " "$h"; curl -s -o /dev/null -w "%{http_code}\n" --max-time 10 "$h/api/health"
done
```
A `200` from either is enough — use that host for the rest of the run. If **both** fail the user is almost certainly off Tailscale; ask them to connect, or ask them to paste the dashboard JSON instead.

Once these are in place, move on — no need to re-check them in the same session unless something fails.

### 1. Get the dashboard

Accept whichever of these the user gives you, in this order of preference:

**A Grafana URL** — the `/d/<uid>/<slug>` in it gives you the UID directly. This is the easiest thing to ask for: "paste the link to the dashboard from your browser." Either hostname is fine; use whichever they sent.

**A dashboard name** — find the UID by searching (`$GRAFANA` is the endpoint you settled on in Step 0):
```bash
curl -s "$GRAFANA/api/search?query=<name>&type=dash-db"
```
If several dashboards match, show the user the titles and folders and let them pick — don't guess.

**Pasted JSON** — check it parses and looks like a dashboard (a `title`, usually `panels`, often `tags`). An "Export for sharing externally" copy is wrapped in a top-level `dashboard` key; unwrap it and mention that you did. If it's malformed, ask for a corrected export rather than patching it up yourself.

Dashboards can be large. When fetching, write straight to the file rather than printing the JSON.

### 2. Locate the repo and get it up to date

Find the local clone (check the current working directory first; if that's not it, ask the user for the path — don't guess at an absolute path on their machine). Once you're in it:

- Check the working tree. If there are uncommitted changes, tell the user and ask how they want to handle it rather than stashing or discarding anything silently. Pre-existing untracked files unrelated to dashboards are fine to leave alone — just stage only the dashboard files yourself, never `git add -A`.
- Note which branch you're on. If it isn't the default branch, say so — the user probably wants this off `main`, not stacked on unrelated work.
- `git checkout main` and `git pull` so the new branch starts from the latest history.

### 3. Check whether this dashboard is already provisioned

Before choosing a destination, search the repo for the dashboard's UID:
```
grep -rl "<uid>" files/monitoring/
```

- **Found under `dashboards/json/`** → this dashboard is Terraform-managed and read-only in the UI. Do **not** also write it to `ui-dashboards/`; that creates two rival copies. Update the existing file in place, and tell the user this one is repo-managed so the change ships when the PR merges.
- **Found under `ui-dashboards/`** → this is an update to an existing backup. Read the current file so you can summarize what changed (new panels, changed queries) in a line or two — not a full diff dump.
- **Not found** → new backup, continue to Step 4.

Also check the older YAML dashboards (`files/monitoring/dashboards/*.yaml`) — a few dashboards are still provisioned that way, and a UID-wide `grep` over `files/monitoring/` catches them.

### 4. Decide the file path

For the normal (UI backup) case:

```
files/monitoring/ui-dashboards/<Grafana-Folder>/<slug>.json
```

- **`<Grafana-Folder>`** — the Grafana folder the dashboard lives in, spaces around dashes collapsed: `Bolt - Sui` → `Bolt-Sui`. Get it from `.meta.folderTitle` on the fetch. A dashboard in no folder goes in `General/`. Unlike the provisioned zone, this directory name is not configuration — it's how we record which folder to restore into, so keep it accurate.
- **`<slug>`** — kebab-case of the title, keeping any leading numbering: `6.1 Bolt Metrics - Executive Overview` → `6-1-bolt-metrics-executive-overview`. Drop trailing markers like `(WIP)`. Replace `&` with `and`.

Confirm the destination with the user before writing — something like: "I'll back this up to `ui-dashboards/Bolt-Sui/my-dashboard.json` — it stays fully editable in Grafana, this is just so it's in git. Sound right?"

### 5. Write the file

Always normalize to the same formatting, so diffs show real changes instead of whitespace churn:

```python
import json, subprocess

GRAFANA = "https://grafana.internal.archway.io"   # or https://grafana.pigeon-saury.ts.net

raw = subprocess.run(
    ["curl", "-sf", "--max-time", "30", f"{GRAFANA}/api/dashboards/uid/{uid}"],
    capture_output=True, check=True).stdout
payload = json.loads(raw)
dash = payload["dashboard"]
folder = payload["meta"]["folderTitle"]      # -> directory name

dash.pop("id", None)   # instance-local; restores match on uid, and it only causes conflicts

with open(path, "w", encoding="utf-8") as fh:
    fh.write(json.dumps(dash, indent=2, sort_keys=True, ensure_ascii=False) + "\n")
```

`indent=2`, sorted keys, `ensure_ascii=False`, trailing newline. Keep `uid` (restores key off it) and `version` (it's the edit counter, useful history). Drop `id`.

For pasted JSON, apply the same normalization instead of writing the paste verbatim.

Then sanity-check what you wrote: it parses, `uid` and `title` are what you expected, and the panel count is plausible.

### 6. Branch and commit

- Branch: `dashboard/<slug>`. If it already exists locally or on the remote, ask whether to reuse it (a follow-up to the same dashboard) or pick another name — never silently overwrite someone's in-progress branch.
- Commit message, matching this repo's conventional-commit style:
  - `feat(monitoring): back up <Dashboard Title> dashboard` — new backup
  - `feat(monitoring): update <Dashboard Title> dashboard` — existing file
  - For several dashboards at once, one commit is fine: `feat(monitoring): back up <N> <group> dashboards`
- Stage only the dashboard files by path. Then confirm nothing unrelated got swept in.

### 7. Show the plan before pushing

Before pushing or opening a PR, summarize: branch name, file path(s), add vs. update, and the commit message. This is the checkpoint that catches a wrong folder before it's a public PR. Wait for a clear go-ahead.

### 8. Push and open the PR

Push the branch, then `gh pr create` against `main` with a title matching the commit and a short body:

```markdown
## What
Backs up the "<Dashboard Title>" Grafana dashboard (built in the UI).

## Where
`files/monitoring/ui-dashboards/<Folder>/<filename>.json`

Backup only — not provisioned by Terraform, so the dashboard stays editable in Grafana.
```

For a change to a provisioned dashboard under `dashboards/json/`, say that instead, and note that merging applies it and that the dashboard is read-only in the UI.

Keep the body short — it's a dashboard file, not a change needing detailed rationale, unless the user offers context worth including.

Share the PR URL when it's done. That's the natural end of the task.

## Restoring a dashboard from the repo

A backup is only worth something if it can be restored, so this is the other half of the workflow. Restoring **writes** to Grafana, so unlike reads it needs a token with Editor rights — anonymous access is Viewer only. This is an admin action; walk the user through it rather than doing it unprompted.

```bash
python3 - <<'PY'
import json, subprocess, os
GRAFANA = "https://grafana.internal.archway.io"   # or https://grafana.pigeon-saury.ts.net
path   = "files/monitoring/ui-dashboards/<Folder>/<file>.json"
folder = "<target folder UID>"   # from /api/search?type=dash-folder
body = json.dumps({"dashboard": json.load(open(path)), "folderUid": folder, "overwrite": True})
subprocess.run(["curl", "-sf", "-X", "POST", f"{GRAFANA}/api/dashboards/db",
    "-H", "Authorization: Bearer " + os.environ["GRAFANA_TOKEN"],
    "-H", "Content-Type: application/json", "-d", body], check=True)
PY
```

Because `uid` is preserved, `overwrite: true` restores over the existing dashboard rather than creating a duplicate. Confirm with the user which they want before running it — restoring over a dashboard someone has since edited discards those edits.

## Handling things that go wrong

- **`gh auth login` stalls or the user isn't sure it worked**: re-run `gh auth status` rather than assuming — don't move on until it shows as logged in.
- **Package install needs sudo/admin rights the user doesn't have**: say so plainly and suggest looping in whoever manages their machine, rather than trying workarounds.
- **Grafana unreachable on one endpoint**: try the other (`grafana.internal.archway.io` ↔ `grafana.pigeon-saury.ts.net`) — they're the same instance behind different ingresses, so one can be down while the other works. Only if **both** fail is it a Tailscale problem: ask them to connect, or fall back to pasted JSON.
- **The dashboard is already provisioned** (Step 3 found it under `dashboards/json/`): don't create a second copy in `ui-dashboards/`. Explain that this one is repo-managed and read-only in the UI, and update the existing file.
- **The user wants a UI dashboard "properly managed"**: make sure they understand it becomes read-only in the browser and loses its version history. Get an explicit yes.
- **Push rejected / branch diverged**: don't force-push. Explain what happened and ask how they'd like to proceed.
- **Merge conflicts on `main`**: if pulling fails or the target folder has moved, stop and explain rather than guessing at a resolution.
- **Ambiguous folder match**: ask rather than picking one.
