# Qubes Storage Helper

Convenient, policy-controlled file storage for Qubes OS: a dedicated *storage qube* exports safe, minimal RPC services (list/copy/move/delete/…); *client qubes* use simple commands and an `fzf` TUI browser to work with files and directories inside the storage qube.

- Client CLI: `qubes-storage`
- Client TUI: `qubes-storage-browse` (fzf-based)
- Storage VM RPC services: `user.Storage*` (server-side scripts)
- dom0 policy snippet: allowlisted, least-privilege qrexec rules

> **License:** GPL-3.0-or-later (see bottom)

---

## Table of Contents

1. [Architecture](#architecture)
2. [Prerequisites](#prerequisites)
3. [Install: Storage Qube](#install-storage-qube)
4. [Install: dom0 Policy](#install-dom0-policy)
5. [Install: Client (AppVM / Template)](#install-client-appvm--template)
6. [How it works (Security model)](#how-it-works-security-model)
7. [Usage: `qubes-storage` CLI](#usage-qubes-storage-cli)
8. [Usage: `qubes-storage-browse` TUI](#usage-qubes-storage-browse-tui)
9. [Configuration & Conventions](#configuration--conventions)
10. [Troubleshooting](#troubleshooting)
11. [FAQ](#faq)
12. [License](#license)

---

## Architecture

- **Storage Qube** (e.g., `my-data`)  
  Hosts a **shared base directory**: `/home/user/shared`.  
  Exposes narrowly-scoped qrexec services:
  ```
  user.StorageList  user.StorageLs   user.StorageStat
  user.StorageGet   user.StoragePut
  user.StorageCopy  user.StorageMove
  user.StorageMkdir user.StorageRmdir
  user.StorageDelete
  ```
  Each service validates paths and is jailed to `/home/user/shared`.

- **dom0**  
  Policy explicitly allows a given **client qube** (e.g., `my-untrusted`) to call the storage services in the **storage qube** (e.g., `my-data`). No global allow.

- **Client Qube(s)** (e.g., `my-untrusted`)  
  - `qubes-storage` (CLI) issues qrexec calls (list/pull/push/mv/cp/rm/…)
  - `qubes-storage-browse` (fzf TUI) gives a keyboard-driven browser with previews and persistent help footer.

---

## Prerequisites

### In the **Storage Qube**
- Linux user `user` with home directory containing `/home/user/shared` (created by the install steps below).
- Standard GNU userland: `bash`, `coreutils` (`stat`, `realpath`, `mv`, `cp`, `rm`, `mkdir`, …), `awk`, `sed`, `grep`.

### In **Client Qubes** (and/or their Template)
- `bash`
- `fzf`
- `less`
- `coreutils`, `awk`, `sed`, `grep`
- `file` (for MIME detection in previews; optional but recommended)
- `hexdump` (usually from `bsdmainutils` or `util-linux`; optional)

> If you also want GUI “pull/push” from the CLI script, install `zenity` in the client qube. The TUI browser does **not** require `zenity`.

---

## Install: Storage Qube

1. **Copy the RPC scripts** from this repo/archive into the storage qube (e.g., `my-data`) under `/etc/qubes-rpc/`:

   Place the files in the storage qube:
   ```
   /etc/qubes-rpc/user.StorageList
   /etc/qubes-rpc/user.StorageLs
   /etc/qubes-rpc/user.StorageStat
   /etc/qubes-rpc/user.StorageGet
   /etc/qubes-rpc/user.StoragePut
   /etc/qubes-rpc/user.StorageCopy
   /etc/qubes-rpc/user.StorageMove
   /etc/qubes-rpc/user.StorageMkdir
   /etc/qubes-rpc/user.StorageRmdir
   /etc/qubes-rpc/user.StorageDelete
   ```

   Make them executable:
   ```bash
   sudo chmod 0755 /etc/qubes-rpc/user.Storage*
   ```

2. **Create the base directory**:
   ```bash
   mkdir -p /home/user/shared
   chown -R user:user /home/user/shared
   chmod 0755 /home/user/shared
   ```

3. **What the server scripts do (summary)**:
   - **Path jail** to `/home/user/shared` using `realpath -m` and string-prefix checks.
   - `StorageList`/`StorageLs` list directory entries (short/long).
   - `StorageStat` returns `stat` for files/dirs.
   - `StorageGet` streams a single file to the client.
   - `StoragePut` writes/overwrites a file (size-capped via `MAX_BYTES`, default 100 MiB; adjust via env if desired).
   - `StorageCopy` copies **files or directories** (recursive for dirs).
   - `StorageMove` moves/renames **files or directories** (atomic within FS).
   - `StorageMkdir` / `StorageRmdir` create/remove directories.
   - `StorageDelete` deletes files.

   > If you change `BASEPATH`, update both server and client to match.

---

## Install: dom0 Policy

Create a dedicated policy file (Qubes 4.1/4.2 style) in dom0, e.g.:

`/etc/qubes/policy.d/50-qubes-storage.policy`
```text
# Allow my-untrusted to call storage services in my-data
user.StorageCopy   *  my-untrusted  my-data  allow
user.StorageGet    *  my-untrusted  my-data  allow
user.StorageLs     *  my-untrusted  my-data  allow
user.StorageMove   *  my-untrusted  my-data  allow
user.StorageStat   *  my-untrusted  my-data  allow
user.StorageDelete *  my-untrusted  my-data  allow
user.StorageList   *  my-untrusted  my-data  allow
user.StorageMkdir  *  my-untrusted  my-data  allow
user.StoragePut    *  my-untrusted  my-data  allow
user.StorageRmdir  *  my-untrusted  my-data  allow
```

Replace `my-untrusted` and `my-data` with **your** client and storage VM names.  
No restart needed; policy applies immediately to new qrexec calls.

---

## Install: Client (AppVM / Template)

1. **Install the CLI `qubes-storage`** into a directory in `PATH`
   (e.g. `/usr/local/bin` in a TemplateVM, or `~/bin` in an AppVM):

   - Set the storage VM name inside the script:
     ```bash
     STORAGE_VM="my-data"
     ```
   - Ensure it’s executable:
     ```bash
     chmod 0755 /usr/local/bin/qubes-storage
     ```

   The script keeps a **current working directory** inside the storage VM in a state file:
   ```
   ~/.cache/qubes-storage-path
   ```
   Default is `"."`. It updates only when `cd` succeeds (i.e., path exists in the storage VM).

2. **Install the TUI `qubes-storage-browse`** (fzf-based):
   ```bash
   chmod 0755 /usr/local/bin/qubes-storage-browse
   ```
   The browser reads the storage VM name directly from the installed `qubes-storage` script by parsing the `STORAGE_VM="..."` line (no extra config is required). It maintains a small cache under:
   ```
   ~/.cache/qubes-storage-browse/
   ```

3. **Packages (client):**
   ```bash
   sudo dnf install -y fzf less file coreutils grep sed gawk util-linux
   # optional for hex preview:
   sudo dnf install -y util-linux   # provides hexdump on most distros
   # optional GUI helpers (CLI only):
   sudo dnf install -y zenity
   ```

---

## How it works (Security model)

- All storage operations run **inside the storage qube**, restricted to `/home/user/shared` via `realpath -m` + strict prefix checks. Any path that would escape (e.g., `../../..`) is rejected.
- `StoragePut` can enforce a max upload size (default `100 MiB` via `MAX_BYTES`).
- `mv` and `cp` support **files and directories** (recursive copy for dirs).
- dom0 policy explicitly ties **source VM** → **target VM** per service, no global wildcards.
- Client tooling never executes code from the storage qube; it only sends paths/data to the RPC services.

---

## Usage: `qubes-storage` CLI

Run `qubes-storage --help` to see the commands. The key commands are:

```
qubes-storage ls [directory]      # short listing
qubes-storage ll [directory]      # long listing
qubes-storage pull <remote> [local]
qubes-storage push <local>  [remote]
qubes-storage rm <remote-file>
qubes-storage cp <source> <dest>  # files or directories
qubes-storage mv <source> <dest>  # rename/move (files or directories)
qubes-storage mkdir <dirname>
qubes-storage rmdir <dirname>
qubes-storage stat <remote>
qubes-storage cd <directory>
qubes-storage pwd
qubes-storage edit <remote-file>  # pulls to a temp file, opens $EDITOR, pushes back
# optional GUI helpers if zenity present:
qubes-storage pull-gui
qubes-storage push-gui
```

**Path semantics**

- Relative paths are relative to the current storage-VM working dir (tracked in `~/.cache/qubes-storage-path`).
- `cd` only updates the cache if the target directory exists in the storage qube.
- `pwd` prints the current remote directory (e.g., `Remote directory: a/b`).

**Examples**

```bash
# list root of storage
qubes-storage ls
# create and enter a folder
qubes-storage mkdir docs
qubes-storage cd docs
# upload a local file
qubes-storage push ~/notes.txt notes.txt
# get file details
qubes-storage stat notes.txt
# rename/move (same command)
qubes-storage mv notes.txt 2025/notes-2025-10-04.txt
# copy a directory recursively
qubes-storage cp templates  archive/templates-backup
# remove an empty directory
qubes-storage rmdir templates
```

---

## Usage: `qubes-storage-browse` TUI

Launch:
```bash
qubes-storage-browse
```

You’ll see a prompt like:
```
[my-data] remote:a/b >
```
…and an `fzf` list of entries. A preview pane at the bottom can show either stats/content or a persistent commands footer.

**Basics**
- **Enter**: enter directory / edit file (opens with the `qubes-storage edit` workflow).
- `..` takes you up one directory.

**Persistent preview/footer (toggle)**
- **Ctrl+H**: show *footer* (commands) in the preview pane (persists).
- **Ctrl+G**: return to *stats/content* preview (persists across actions and directory changes).

**File preview**
- Shows MIME type and the first *N* lines (default `200`) of text files; for binary files, a short hex dump.  
  `PREVIEW_MAX_LINES` and `PREVIEW_MAX_BYTES` are tunable via environment variables.

**View full file**
- **Ctrl+F**: open the **full** file content in `less`.  
  The browser maintains a small local cache; it automatically **re-pulls** the file whenever size or modification time changes in the storage qube (fingerprint check), so previews and views stay fresh after edits/pushes.

**Navigation & actions (default bindings)**

- **Ctrl+U**: up (`cd ..`)
- **Ctrl+J**: jump to a remote path (`cd <path>`)
- **Ctrl+N**: create directory (`mkdir`)
- **Ctrl+R**: rename/move (`mv`) — works for files **and** directories
- **Ctrl+Y**: copy (`cp`) — works for files **and** directories
- **Ctrl+D**: delete (file → `rm`; dir → `rmdir`) — with confirmation
- **Ctrl+P**: pull selected **file** to the current local directory
- **Ctrl+W**: push a local file (prompt for local path) into the current remote directory
- **Ctrl+S**: show `stat` in `less`
- **Ctrl+O**: open full help in `less`
- **Esc**: quit

> The TUI reads the storage VM name by parsing the `STORAGE_VM="..."` line from your installed `qubes-storage` script (no separate config file needed). The prompt shows it as `[VM] remote:<path> >`.

---

## Configuration & Conventions

- **Storage VM base path:** `/home/user/shared` (server-side).  
  If you change it, update the RPC scripts accordingly and re-install.
- **Client working dir state:** `${XDG_CACHE_HOME:-$HOME/.cache}/qubes-storage-path`
- **TUI cache:** `${XDG_CACHE_HOME:-$HOME/.cache}/qubes-storage-browse/`
- **TUI preview tuning (set as env vars if you like):**
  ```bash
  PREVIEW_MAX_LINES=300 PREVIEW_MAX_BYTES=$((2*1024*1024)) qubes-storage-browse
  ```

---

## Troubleshooting

**“Permission denied” on qrexec / service not found**  
- Verify dom0 policy rules (typos, wrong VM names).
- Ensure `/etc/qubes-rpc/user.Storage*` exist in the **storage VM** and are **executable**.

**`cd` appears to “forget” the directory**  
- `cd` only updates the client cache if the directory **exists** on the storage side. Check with:
  ```bash
  printf '%s\n' "<dir>" | qrexec-client-vm my-data user.StorageStat
  ```

**TUI preview doesn’t match edited file**  
- The browser auto-refreshes previews via a fingerprint check (size + modify timestamp). If you still observe stale content, ensure:
  - `stat` output in the storage VM includes `Size:` and `Modify:` lines (GNU `stat`).
  - Client and storage qube clocks are sane.

**Enter triggers wrong action**  
- In `fzf`, `Enter` equals `Ctrl+M`. The TUI does **not** bind `Ctrl+M` to other actions; if you customize bindings, avoid `Ctrl+M`.

**Binary file previews look noisy**  
- Install `util-linux` (for `hexdump`) to get a compact hex dump; or press **Ctrl+F** to view the file in `less` instead.

---

## FAQ

**Q: Why are “rename” and “move” a single command?**  
Because on Unix, `mv` already does both: rename within a directory or move across directories. The server’s `StorageMove` supports files **and** directories.

**Q: Can I restrict which client qubes may call the storage services?**  
Yes — that’s exactly what the dom0 policy does. Use per-VM rules; don’t use `@anyvm` unless you really mean it.

**Q: Can I change the base directory?**  
Yes. Adjust `BASEPATH` in the storage scripts and re-deploy. Keep the `realpath`/prefix checks.

---

## License

This project is released under the **GNU General Public License v3.0 or later** (GPLv3+).

```
Qubes Storage Helper
Copyright (C) 2025  one7two99

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published
by the Free Software Foundation, version 3 of the License.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <https://www.gnu.org/licenses/>.
```
