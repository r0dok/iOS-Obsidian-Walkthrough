# Obsidian iOS Git Sync via iSH

Getting Obsidian to sync with GitHub on iOS is not straightforward. The plugin behaves differently on mobile, iSH has memory constraints, and there are a few failure modes that will cost you time if you don't know about them upfront. This is a practical walkthrough of what actually works.

---

## What you need

- Obsidian installed on iOS
- [iSH Shell](https://apps.apple.com/app/ish-shell/id1436902243) installed
- A GitHub account with a private repo for your notes
- A GitHub Personal Access Token (PAT) with `repo` scope

---

## How it works

The Obsidian Git plugin on iOS does **not** use a system git binary. It uses isomorphic-git, a JavaScript implementation bundled in the plugin. This means:

- SSH does not work. HTTPS only.
- The "Custom Git binary path" setting in the plugin does nothing on iOS.
- Auth is done via username + PAT, either in the plugin settings or embedded in the remote URL.

The initial clone is done through **iSH** rather than the plugin, because the plugin's clone tends to crash on large vaults due to iOS memory limits. iSH handles it more reliably with `--depth=1`.

---

## Setup

### 1. Install git in iSH

```sh
apk add git
```

### 2. Mount your Obsidian vault folder

In iSH, tap the settings icon > Mounts > add a folder. Navigate to your Obsidian vault location under "On My iPhone". This makes the vault accessible from iSH at a path like `/root/obsidian`.

> Do not use iCloud Drive as your vault location. It conflicts with iSH file writes.

### 3. Clone your repo

```sh
cd /root/obsidian
git clone --depth=1 https://USERNAME:YOUR_PAT@github.com/USERNAME/REPO.git .
```

The `--depth=1` flag fetches only the latest snapshot. Without it, large repos will exhaust iOS memory mid-clone and corrupt the git object store.

### 4. Set the upstream branch

Your local branch needs to track the remote:

```sh
git push --set-upstream origin main
```

If your local branch is named `master` but the remote uses `main`:

```sh
git branch -m master main
git push --set-upstream origin main
```

### 5. Open the vault in Obsidian

Obsidian > Open folder as vault > navigate to the same folder you cloned into.

### 6. Install and configure the Obsidian Git plugin

Community plugins > Browse > search "obsidian-git" > Install > Enable.

In the plugin settings:

- **Username**: your GitHub username
- **Password/Personal access token**: your PAT

If the plugin still fails to push (TLS errors, missing remote errors), set the remote URL with credentials embedded directly in iSH instead:

```sh
git remote set-url origin https://USERNAME:YOUR_PAT@github.com/USERNAME/REPO.git
```

This bypasses the plugin's auth layer entirely.

### 7. Disable auto-sync

The plugin's background sync causes freezes on large vaults. In plugin settings:

- Vault backup interval: `0`
- Auto pull interval: `0`
- Pull on startup: off

Sync manually via the command palette.

---

## Daily use

**Before editing on iOS:**
Command palette > `Git: Pull`

**After editing on iOS:**
Command palette > `Git: Commit-and-sync`

**On desktop before editing:**
```sh
git pull
```

**On desktop after editing:**
```sh
git add -A && git commit -m "notes" && git push
```

---

## Known issues

**TLS errors on clone** — use the PAT-embedded URL format instead of relying on the plugin's auth fields.

**"No upstream branch is set"** — run `git push --set-upstream origin main` from iSH.

**"remote OR url parameter not provided"** — the plugin lost the remote URL. Fix with `git remote set-url` in iSH.

**Corrupt git objects after clone** — iSH ran out of memory. Delete the folder and reclone with `--depth=1`.

**App freezes** — disable auto-sync intervals in plugin settings. Sync manually only.

**Branch mismatch** — if local is `master` and remote is `main`, rename: `git branch -m master main`.

---

## What not to do

- Do not share screenshots of your PAT. Revoke it immediately if you do.
- Do not store the vault on iCloud Drive.
- Do not edit on two devices simultaneously without pulling first. The plugin does not handle merge conflicts.
