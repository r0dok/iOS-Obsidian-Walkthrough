# Obsidian iOS Git Sync via Codeberg

No SSH, no iSH for daily use. The Obsidian Git plugin handles everything over HTTPS with a PAT. iSH is only needed once — for the initial clone and remote setup. After that, forget it exists.

---

## What you need

- Obsidian installed on iOS
- [iSH Shell](https://apps.apple.com/app/ish-shell/id1436902243) installed
- A Codeberg account with a private repo for your notes
- A Codeberg Personal Access Token with `repository` read/write scope

---

## How it works

The Obsidian Git plugin on iOS does **not** use a system git binary. It uses isomorphic-git, a JavaScript implementation bundled in the plugin. This means:

- SSH does not work. HTTPS only.
- The "Custom Git binary path" setting does nothing on iOS.
- Auth is done via username + PAT, either in plugin settings or embedded in the remote URL.

The initial clone is done through iSH rather than the plugin because the plugin's clone tends to crash on large vaults due to iOS memory limits. iSH handles it more reliably with `--depth=1`.

---

## Setup

### 1. Generate a Codeberg PAT

Codeberg → Settings → Applications → Access tokens → Generate new token.

- Name it something like `obsidian_ios`
- Permissions: `repository` → Read and Write
- Copy the token immediately, it won't show again

### 2. Create your notes repo on Codeberg

If it doesn't exist yet, create a private repo. If you're migrating from GitHub, push your existing repo to Codeberg first from a desktop machine:

```bash
git remote set-url origin https://USERNAME:YOUR_PAT@codeberg.org/USERNAME/notes.git
git push
```

### 3. Install git in iSH

```
apk add git
```

### 4. Mount your Obsidian vault folder

```
cd ~
mkdir obsidian
mount -t ios . obsidian
```

This triggers the iOS file picker. Select your Obsidian vault folder from "On My iPhone". The vault is now accessible from iSH at `~/obsidian`.

> Do not use iCloud Drive as your vault location. It conflicts with iSH file writes.

### 5. Clone your repo

```
cd /root/obsidian
git clone --depth=1 https://USERNAME:YOUR_PAT@codeberg.org/USERNAME/notes.git .
```

The `--depth=1` flag fetches only the latest snapshot. Without it, large repos will exhaust iOS memory mid-clone and corrupt the git object store.

If you already have vault files in the folder (e.g. migrating from GitHub), skip the clone. Instead:

```bash
git init
git remote add origin https://USERNAME:YOUR_PAT@codeberg.org/USERNAME/notes.git
git fetch --depth=1 origin main
git reset --hard origin/main
```

### 6. Set the upstream branch

```
git push --set-upstream origin main
```

If your local branch is named `master` but the remote uses `main`:

```
git branch -m master main
git push --set-upstream origin main
```

### 7. Open the vault in Obsidian

Obsidian → Open folder as vault → navigate to the same folder you cloned into.

### 8. Install and configure Obsidian Git plugin

Community plugins → Browse → search "obsidian-git" → Install → Enable.

In the plugin settings:

- **Username**: your Codeberg username
- **Password/Personal access token**: your Codeberg PAT

If the plugin still fails to push, set the remote URL with credentials embedded directly in iSH:

```
git remote set-url origin https://USERNAME:YOUR_PAT@codeberg.org/USERNAME/notes.git
```

You can also do this from within the Obsidian command palette: `Git: Edit remotes`.

### 9. Disable auto-sync

The plugin's background sync causes freezes on large vaults. In plugin settings:

- Vault backup interval: `0`
- Auto pull interval: `0`
- Pull on startup: off

Sync manually via the command palette.

---

## Daily use

Everything happens inside Obsidian. iSH is not part of daily use.

**Before editing on iOS:**
Command palette → `Git: Pull`

**After editing on iOS:**
Command palette → `Git: Commit-and-sync`

**On desktop before editing:**

```bash
git pull
```

**On desktop after editing:**

```bash
git add -A && git commit -m "notes" && git push
```

---

## If the repo gets into a broken state

This can happen if you interrupted a clone, switched remotes mid-operation, or have a merge conflict from diverged histories. Reset cleanly from iSH:

```bash
cd ~/obsidian
git fetch origin
git reset --hard origin/main
```

This discards any local-only changes and syncs to whatever is on Codeberg. If you have unsaved notes you care about, copy them out of the vault folder first.

---

## Known issues

**"refusing to merge unrelated histories"** — happens when migrating from GitHub to Codeberg if the remote history doesn't match the local clone. Fix:

```bash
git pull --depth=1 origin main --allow-unrelated-histories
```

If there are conflicts, they'll likely be in `.obsidian/workspace.json` or `workspace-mobile.json` — safe to take the remote version:

```bash
git checkout --theirs .obsidian/workspace.json
git checkout --theirs .obsidian/workspace-mobile.json
git add .obsidian/workspace.json .obsidian/workspace-mobile.json
git reset --hard origin/main
```

**"No upstream branch is set"** — run `git push --set-upstream origin main` from iSH.

**"remote OR url parameter not provided"** — the plugin lost the remote URL. Fix with `git remote set-url` in iSH or via `Git: Edit remotes` in the command palette.

**Corrupt git objects after clone** — iSH ran out of memory. Delete the folder contents and reclone with `--depth=1`.

**App freezes during sync** — disable auto-sync intervals in plugin settings. Sync manually only.

**Branch mismatch** — if local is `master` and remote is `main`: `git branch -m master main`.

**git commit hangs in iSH** — git is waiting for an editor. Either use `git commit --no-edit` or set `git config --global core.editor "true"` to suppress it.

---

## What not to do

- Do not share screenshots of your PAT. Revoke it immediately if you do.
- Do not store the vault on iCloud Drive.
- Do not edit on two devices simultaneously without pulling first. The plugin does not handle merge conflicts gracefully.
- Do not run `git add -A` on a 100MB+ vault in iSH unless you have time to kill. The plugin handles incremental syncs faster.