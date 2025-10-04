# Installation Guide

This document describes how to install the Qubes Storage Helper components:
- Storage Qube RPC services
- dom0 qrexec policy
- Client-side tools (`qubes-storage`, `qubes-storage-browse`)
- Required packages

> **Assumptions**
> - Storage qube: `my-data`
> - Client qube: `my-untrusted`
> - Base path in storage qube: `/home/user/shared`

Adjust names as needed.

---

## 1) Prepare the Storage Qube (`my-data`)

1. **Copy RPC scripts into the storage qube** (from this repo/archive):
   Place the following files under `/etc/qubes-rpc/` inside the storage qube:
   ```text
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

2. **Permissions**
   ```bash
   sudo chmod 0755 /etc/qubes-rpc/user.Storage*
   ```

3. **Create the shared base directory**
   ```bash
   mkdir -p /home/user/shared
   chown -R user:user /home/user/shared
   chmod 0755 /home/user/shared
   ```

4. **(Optional) Set upload size limit for `StoragePut`**
   The server script can honor an environment variable `MAX_BYTES`. Default is 100 MiB.
   Edit `user.StoragePut` if you want to change the default.

---

## 2) Configure dom0 Policy

Create `/etc/qubes/policy.d/50-qubes-storage.policy` in **dom0**:

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

> Replace `my-untrusted` and `my-data` with your VM names. No restart is required.

---

## 3) Install Client Tools (TemplateVM or AppVM)

You can install into a TemplateVM for persistence across derived AppVMs, or directly into a single AppVM.

1. **Dependencies**
   ```bash
   sudo dnf install -y fzf less file coreutils grep sed gawk util-linux
   # optional for GUI helpers (CLI only)
   sudo dnf install -y zenity
   ```

2. **Install `qubes-storage`**
   Place the script in `PATH` (e.g., `/usr/local/bin/qubes-storage`) and make executable:
   ```bash
   sudo install -m 0755 qubes-storage /usr/local/bin/qubes-storage
   ```
   Edit the script and set:
   ```bash
   STORAGE_VM="my-data"
   ```
   The current remote path is tracked in:
   ```text
   ~/.cache/qubes-storage-path
   ```

3. **Install `qubes-storage-browse` (fzf TUI)**
   ```bash
   sudo install -m 0755 qubes-storage-browse /usr/local/bin/qubes-storage-browse
   ```
   The browser parses the storage VM name from the installed `qubes-storage` script (no extra config). It keeps a small cache in:
   ```text
   ~/.cache/qubes-storage-browse/
   ```

---

## 4) Verification

From the **client qube** (`my-untrusted`):

```bash
# quick connectivity check (lists storage root)
qubes-storage ls

# create a test dir and navigate
qubes-storage mkdir demo && qubes-storage cd demo && qubes-storage pwd

# upload a file
echo "hello" > /tmp/hello.txt
qubes-storage push /tmp/hello.txt hello.txt
qubes-storage stat hello.txt

# TUI browser
qubes-storage-browse
```

From the **storage qube** (`my-data`), confirm the file exists:
```bash
ls -l /home/user/shared/demo/hello.txt
```

---

## 5) Upgrades

- Replace scripts in:
  - **storage qube:** `/etc/qubes-rpc/user.Storage*`
  - **client:** `/usr/local/bin/qubes-storage`, `/usr/local/bin/qubes-storage-browse`
- Keep permissions executable (`chmod 0755 ...`).
- No policy changes are needed unless you add new RPC names.

---

## 6) Uninstall

- **Client qube:**
  ```bash
  sudo rm -f /usr/local/bin/qubes-storage /usr/local/bin/qubes-storage-browse
  rm -rf ~/.cache/qubes-storage-browse ~/.cache/qubes-storage-path
  ```

- **Storage qube:**
  ```bash
  sudo rm -f /etc/qubes-rpc/user.Storage*
  # optional: keep or remove /home/user/shared
  ```

- **dom0:**
  ```bash
  sudo rm -f /etc/qubes/policy.d/50-qubes-storage.policy
  ```

---

## Notes & Tips

- **Directory-only `cd`**: the client updates its remote-path cache only if the target directory exists in the storage qube.
- **mv/cp directories**: supported on server side; copy is recursive, move is atomic within the same filesystem.
- **Preview tuning** (TUI):
  ```bash
  PREVIEW_MAX_LINES=300 PREVIEW_MAX_BYTES=$((2*1024*1024)) qubes-storage-browse
  ```
- **Multiple client VMs**: add specific dom0 policy lines per client VM.
- **Security**: all server scripts jail operations under `/home/user/shared` with `realpath -m` and prefix checks. Do not remove these guards.
