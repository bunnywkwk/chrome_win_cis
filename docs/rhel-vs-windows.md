# chrome_win_cis vs chrome_cis: RHEL and Windows compared

Same benchmark (CIS Google Chrome v3.0.0), same rules, same role logic. What differs: **where the policy is stored**
(JSON file on RHEL, registry on Windows) and **how Ansible reaches the machine** (SSH vs WinRM).

Status: `chrome_cis` (RHEL) is tested on VMs, containers and CI. `chrome_win_cis`: one first run on the win11 VM
with the earlier version (before discovery and gates: install + policies, `failed=0`,
`~/chrome_win_cis_test/runs/runs-win11-first.log`); the current version is not run yet and `chrome://policy` is not
checked yet.

## 0. How the Windows test was set up and run

### Windows VM (win11, 192.168.20.60, Proxmox)

| Step | Command / setting (on the VM, PowerShell as Administrator) | Why |
|---|---|---|
| Network card | Proxmox NIC model **E1000E** instead of VirtIO | Windows has no VirtIO driver out of the box, so the VM had no network |
| Network profile | `Set-NetConnectionProfile -NetworkCategory Private` | WinRM setup refuses to open the firewall on a "Public" network |
| Allow ping | firewall rule for ICMPv4 inbound | Windows blocks ping by default; used to check the VM is reachable |
| Turn on WinRM | `Enable-PSRemoting -Force` | starts the WinRM service and its HTTP listener on port 5985 |
| Open port 5985 | firewall rule for WinRM HTTP (TCP 5985) inbound | so the control node can connect |
| Check | `winrm enumerate winrm/config/listener` | shows `Transport = HTTP`, `Port = 5985` |
| Snapshot | Proxmox snapshot after `win_ping` worked | rollback point for tests |

Login: local Administrator account `frqadmin`, password typed at run time (`-k`), not stored.

### Control node (workstation)

| Item | Value | Why |
|---|---|---|
| Ansible | `~/.ansible-2.16.1-env` (ansible-core 2.16.19), same as the RHEL role | one control node for both roles |
| Python library | `pip install pywinrm` (0.5.0) in that venv | Ansible's WinRM connection needs it (SSH needs nothing extra) |
| Collection | `ansible.windows` 2.8.0 (`requirements.yml`: `>=2.5.0,<3.0.0`), installed with `ansible-galaxy collection install -r ~/ansible-cis/chrome_win_cis/requirements.yml -p ./collections` | the `win_*` modules |

### Test project `~/chrome_win_cis_test`

| File | Content |
|---|---|
| `ansible.cfg` | `inventory = sysconfig/inventory.yml`, `roles_path = ~/ansible-cis` (role used from the local folder), `collections_path = ./collections` |
| `sysconfig/inventory.yml` | group `windows`, host `win11` 192.168.20.60; `ansible_connection: winrm`, `ansible_winrm_transport: ntlm`, `ansible_port: 5985`, `ansible_user: frqadmin` |
| `sysconfig/group_vars/windows/chrome.yml` | site settings (install, levels, site values) |
| `playbooks/site.yml` | `hosts: windows`, `roles: [chrome_win_cis]` (no `become`) |

### Run

```bash
cd ~/chrome_win_cis_test
source ~/.ansible-2.16.1-env/bin/activate
ansible windows -m ansible.windows.win_ping -k                          # connection test: "pong"
ansible-playbook playbooks/site.yml -k | tee runs/<name>.log            # harden
```

Terms to use: **WinRM** (not SSH), **NTLM** authentication (Windows username/password), **port 5985** (WinRM over
HTTP; 5986 would be HTTPS), **pywinrm** (Python client for WinRM on the control node), **ansible.windows**
collection (the `win_*` modules, which run as PowerShell on the target).

## 1. Terms

| Windows term | Meaning | RHEL counterpart |
|---|---|---|
| Registry | Windows settings database (`regedit`) | the file system under `/etc` |
| Hive `HKLM` (`HKEY_LOCAL_MACHINE`) | settings for the whole machine (`HKCU` = one user) | `managed` policy directory (applies to all users) |
| Key | a "folder" in the registry, e.g. `HKLM\SOFTWARE\Policies\Google\Chrome` | policy directory `/etc/opt/chrome/policies/managed/` |
| Subkey | a key inside a key | (none; a JSON array is used) |
| Value (name + data + type) | one policy, e.g. `CloudPrintProxyEnabled = 0` | one key/value pair in `chrome_cis.json` |
| `REG_DWORD` | number; also used for true/false (1/0) | JSON number / `true` / `false` |
| `REG_SZ` | text | JSON string |
| Group Policy (GPO), ADMX templates | the normal Windows admin tool; Google ships ADMX files so Chrome policies appear in the GPO editor. CIS audit steps are written as GPO paths | (none) |
| WinRM | Windows remote management, port 5985 (HTTP), NTLM login | SSH, port 22 |
| Enterprise MSI | Google's Windows installer for companies | RPM from Google's repo / EPEL |

The role writes the registry values directly. The result is the same as a GPO with Google's ADMX templates.

## 2. Side by side

| | RHEL (`chrome_cis`) | Windows (`chrome_win_cis`) |
|---|---|---|
| Chrome policy location | `/etc/opt/chrome/policies/managed/chrome_cis.json` | `HKLM\SOFTWARE\Policies\Google\Chrome` |
| Chromium policy location | `/etc/chromium/policies/managed/chrome_cis.json` | `HKLM\SOFTWARE\Policies\Chromium` |
| List policy, e.g. `["*"]` | JSON array | subkey `<Policy>` with values `1`, `2`, `3` ... |
| Connection | SSH key, `become: true` (root) | WinRM + NTLM, password (`-k`), user is a local Administrator |
| Runs on the target | Python | PowerShell (no Python) |
| Modules | `ansible.builtin` (`package_facts`, `dnf`, `copy`) | `ansible.windows` (`win_stat`, `win_find`, `win_package`, `win_regedit`, `win_reg_stat`) |
| Browser found by | RPM names (`google-chrome-*`, `chromium`) | `chrome.exe` in Program Files |
| Version read from | RPM version | version folder next to `chrome.exe`, e.g. `Application\155.0.8059.39` |
| Install | Chrome (exact version possible) or Chromium (EPEL) | Chrome only, newest MSI only |
| Gated rules (D17 ranges) | 8 | 10 (also 1.19 and 3.6: they existed on Windows) |
| Windows-only 2.8.2, 2.10.1, 2.30, 3.13 | off (`false`) | normal rules |
| Google Update 2.1.1, 2.1.2 | off (no Google Update on Linux) | applied to `HKLM\SOFTWARE\Policies\Google\Update`; CIS: only works on domain-joined machines |
| Rule switched off later | removed (file rewritten every run) | **stays** until deleted by hand |
| Check by hand | `sudo cat /etc/opt/chrome/policies/managed/chrome_cis.json` | `Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Google\Chrome'` |
| In the browser | `chrome://policy`, Source "Platform" | same |

## 3. Files: what each does, and why it differs from RHEL

### `vars/main.yml`

| Var | Does | RHEL | Why different |
|---|---|---|---|
| `chrome_win_cis_chrome_key`, `_chromium_key` | registry key per browser | `chrome_cis_policy_dirs` paths | registry instead of folders |
| `chrome_win_cis_update_key` | Google Update key (2.1.1, 2.1.2) | none | Google Update exists only on Windows |
| `chrome_win_cis_chrome_exes`, `_chromium_exes` | where `chrome.exe` can be | none | no package database to ask on Windows |
| `chrome_win_cis_found_exes` | the `chrome.exe` paths that exist | none | |
| `chrome_win_cis_policy_keys` | keys of the browsers found | `chrome_cis_policy_dirs` | same idea |
| `chrome_win_cis_list_policies` / `_value_policies` | splits the policies into lists and single values | none | JSON stores a list in one line; the registry needs a subkey (see `post_list.yml`) |

### `tasks/prelim.yml`

| Task | Does | RHEL |
|---|---|---|
| Install Google Chrome (opt-in) | imports `install.yml` | same, but Chrome or Chromium |
| Find chrome.exe of Chrome and Chromium | `win_stat` on each path | `package_facts` |
| Find the browser version folders | `win_find` for folders named like `155.0.8059.39` next to `chrome.exe` | version comes with `package_facts` |
| Discover installed browsers | sets `chrome_win_cis_chrome_found`, `_chromium_found`, `_browser_major` (lowest major) | same facts |
| Stop when no browser is installed | ends the host cleanly | same |
| Start the policy set | `chrome_win_cis_policies: {}` | same |

Why `chrome.exe` and a folder: Windows has no RPM database. Chrome's installer puts each version in its own folder
next to `chrome.exe`, so the folder name is the version.

### `tasks/install.yml`

One `win_package` task: downloads Google's Enterprise MSI and installs it. `creates_path` = `chrome.exe`, so it is
skipped when Chrome is already there (idempotent). RHEL needs three steps (key, repo, `dnf`) and can pick a version.

### `tasks/section_<n>/cis_*.yml` (the rules)

Identical to RHEL: each rule is a `set_fact` that adds its policy to `chrome_win_cis_policies`, gated by level, rule
toggle, site value and (for removed policies) the version range. Exception: 2.1.1 and 2.1.2 write directly with
`win_regedit` to the Google Update key, because they are not Chrome policies.

### `tasks/post.yml`

| Task | Does | RHEL |
|---|---|---|
| Write the single-value policies | `win_regedit`, one value per policy per browser key. `true/false` → `REG_DWORD 1/0`, text → `REG_SZ`, numbers → `REG_DWORD` | one `copy` of the whole JSON file |
| Write the list policies | runs `post_list.yml` once per list policy per browser key | (inside the same JSON file) |
| Report | browsers found, version, keys, number of policies | same |

`product(...)` pairs every key with every policy: 2 browsers x 70 policies = 140 writes.

### `tasks/post_list.yml` (list policies)

The problem: a JSON file holds a list in one line; the registry cannot. Group Policy stores a list as a **subkey
named after the policy**, with one value per item, named `1`, `2`, `3` ...

Example, 2.3.5 with `["https://a.com", "https://b.com"]`:

```
RHEL  chrome_cis.json:
  "ThirdPartyStoragePartitioningBlockedForOrigins": ["https://a.com", "https://b.com"]

Windows registry:
  HKLM\SOFTWARE\Policies\Google\Chrome\ThirdPartyStoragePartitioningBlockedForOrigins
      1    REG_SZ    https://a.com
      2    REG_SZ    https://b.com
```

The three tasks, run for one list policy in one browser key:

| Task | Does | Why |
|---|---|---|
| Read the current items | `win_reg_stat` lists the values already in the subkey, e.g. `1, 2, 3` | to know what is there before changing it |
| Write the items | `win_regedit` writes item 1 as value `1`, item 2 as value `2` ... (`ansible_loop.index` = position) | the Group Policy format Chrome reads |
| Remove items no longer in the list | deletes value names that are not `1`..`n` | the list got shorter: e.g. it had 3 items, now 2 → value `3` must go, or Chrome would still use it |

A CIS "Disabled" exception list (the rule always writes `[]`, e.g. `HSTSPolicyBypassList`) writes no values and
removes any old ones.
RHEL needs none of this: `copy` rewrites the whole file, so old items disappear by themselves.

How "Remove" triggers: no `when:`; its loop is **names now in the registry minus names that should be there**
(`1`..`n`). Empty result = the task does nothing; each leftover name = one delete.

Worked example, 2.27 `chrome_win_cis_http_allowlist` (`defaults/main.yml`) on one PC:

| Run | group_vars | Now | Should | Remove deletes | Registry after |
|---|---|---|---|---|---|
| 1 | `["intranet.lab.local", "wiki.lab.local", "old-app.lab.local"]` | none | 1, 2, 3 | nothing | 1, 2, 3 |
| 2 | `["intranet.lab.local", "wiki.lab.local"]` | 1, 2, 3 | 1, 2 | `3` | 1, 2 |
| 3 | same as run 2 | 1, 2 | 1, 2 | nothing (`changed=0`) | 1, 2 |

Why it matters: without run 2's delete, `old-app.lab.local` would stay allowed over plain HTTP although the admin
removed it. Check: `reg query HKLM\SOFTWARE\Policies\Google\Chrome\HttpAllowlist` and `chrome://policy`.

Limit: setting the list back to `[]` or switching 2.27 off skips the rule, so `post_list.yml` does not run for
`HttpAllowlist` and values 1, 2 stay (see Open points).

## 4. Open points (Windows)

- Not tested on the VM yet; check `chrome://policy` shows OK for the written policies.
- A rule switched off later keeps its old value (RHEL removes it). Delete it by hand, or add a cleanup later.
- Some Chrome policies only take effect on a domain-joined or cloud-managed Windows machine; check on the VM which
  ones show an error on a standalone install.
- Chromium path `C:\Program Files\Chromium\Application\chrome.exe` (system-wide install) is assumed, not yet seen.
