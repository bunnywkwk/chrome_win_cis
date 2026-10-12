# Build guide: chrome_win_cis

Google Chrome on Windows 10/11, CIS Google Chrome Benchmark v3.0.0. Same rules, IDs, toggles, levels **and rule
tasks** as `chrome_cis` (RHEL). Only `post.yml` (how the policies are written) and the install change.

Status: generated, **not tested**. Written already: `defaults/`, `vars/`, `meta/`, `requirements.yml`, lint
config, empty `tasks/` files. You paste the code from this guide into them.

## 1. How it works

Same idea as Linux:

1. `prelim.yml` starts an empty `chrome_win_cis_policies`.
2. Each rule task adds its policy with `set_fact` + `combine`. The rule files are the Linux files with
   `chrome_cis_` renamed to `chrome_win_cis_`.
3. `post.yml` writes everything at once. On Linux that is one JSON file; on Windows it is the registry key the
   benchmark names in every Audit section: `HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Google\Chrome`.

| | Linux `post.yml` | Windows `post.yml` |
|---|---|---|
| Writes to | `/etc/opt/chrome/policies/managed/chrome_cis.json` | `HKLM:\SOFTWARE\Policies\Google\Chrome` |
| Module | `copy` (one file) | `win_regedit` (one loop over the policies) |
| true / false | `true` / `false` | `REG_DWORD` 1 / 0 |
| number | number | `REG_DWORD` |
| text | string | `REG_SZ` |
| list, e.g. `["*"]` | JSON array | subkey `...\Chrome\<Policy>` with `REG_SZ` values `1`, `2`, `3` ... (benchmark: `ExtensionInstallBlocklist\1 = *`) |
| Shows `changed` | the `copy` task | the `win_regedit` loop items that differ |

`vars/main.yml` splits the collected policies in two: `chrome_win_cis_value_policies` (one registry value each)
and `chrome_win_cis_list_policies` (one subkey each).

Lists need one extra step. If a list had 3 items and now has 2, value `3` would stay and still be applied, so
`post_list.yml` reads the subkey first and removes the numbers that are no longer in the list.

### 1.1 Differences from Linux

| Topic | Linux | Windows |
|---|---|---|
| 2.1.1, 2.1.2 (Google Update) | N/A, comments only | two tasks that write directly to `HKLM:\SOFTWARE\Policies\Google\Update` (a different key, not a Chrome policy). CIS: "available only on Windows instances that are joined to a Microsoft Active Directory domain", so a workgroup VM ignores them |
| 2.8.2, 2.10.1, 2.30, 3.13 | N/A (Windows only) | normal rules (tag `not_applicable` replaced by their topic) |
| Rule switched off later | removed (file rebuilt) | value stays; delete it with `reg delete` (the role does not delete values it did not write this run, because other tools or GPOs can use the same key) |
| Install | dnf repo, channel, exact version | Google Enterprise MSI, newest only |
| Browser | chrome / chromium | chrome only |

### 1.2 Things to check on the VM

- Chrome ignores some policies on machines not joined to a domain. `chrome://policy` shows those with a warning.
- Chrome rereads registry policies every few hours. After a run, click **Reload policies** on `chrome://policy`.
- The MSI URL and `win_package` with a URL come from Google's and Ansible's docs, not tried yet.

## 2. Test project

The VM is ready (WinRM, `win_ping` = pong, snapshot `pre-chrome`). Lab prep problems: `troubleshooting.md`
T-W1 to T-W3.

`inventory.ini`:

```ini
[windows]
win11 ansible_host=192.168.20.60

[windows:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_port=5985
ansible_user=frqadmin
```

Password: not stored; run `ansible-playbook` with `-k` and type it.

`playbooks/site.yml`:

```yaml
---
- name: Harden Google Chrome (CIS) on Windows
  hosts: windows
  roles:
    - role: chrome_win_cis
```

`group_vars/windows/main.yml`:

```yaml
---
chrome_win_cis_install: true
chrome_win_cis_level_1: true
chrome_win_cis_level_2: true
chrome_win_cis_rule_4_7: false          # no DoH server in the lab
```

## 3. Files to paste

Order: `main.yml` → `prelim.yml` → `install.yml` → sections → `post.yml` → `post_list.yml`.

### 3.1 `tasks/main.yml`

What: prelim, install, five sections, post. Why: same as Linux; prelim and post are tagged `always` so
`--tags rule_x` runs still check the target and write.

```yaml
---
- name: PRELIM | Checks, Chrome detection, start the policy set
  ansible.builtin.import_tasks:
    file: prelim.yml
  tags:
    - always

- name: INSTALL | Google Chrome (opt-in)
  when: chrome_win_cis_install
  ansible.builtin.import_tasks:
    file: install.yml

- name: SECTION 1 | Enforced Defaults
  when: chrome_win_cis_section1
  ansible.builtin.import_tasks:
    file: section_1/main.yml

- name: SECTION 2 | Attack Surface Reduction
  when: chrome_win_cis_section2
  ansible.builtin.import_tasks:
    file: section_2/main.yml

- name: SECTION 3 | Privacy
  when: chrome_win_cis_section3
  ansible.builtin.import_tasks:
    file: section_3/main.yml

- name: SECTION 4 | Data Loss Prevention
  when: chrome_win_cis_section4
  ansible.builtin.import_tasks:
    file: section_4/main.yml

- name: SECTION 5 | Forensics
  when: chrome_win_cis_section5
  ansible.builtin.import_tasks:
    file: section_5/main.yml

- name: POST | Write the policies and report
  ansible.builtin.import_tasks:
    file: post.yml
  tags:
    - always
```

### 3.2 `tasks/prelim.yml`

What: checks Ansible and the OS, finds `chrome.exe`, stops when Chrome is missing and install is off, starts the
policy set. Why: same as Linux. MSI installs have no `package_facts`, so the role looks for the executable;
`chrome_win_cis_installed` in `vars/main.yml` turns the result into its path, or `''`.

```yaml
---
- name: "PRELIM | AUDIT | Check ansible-core version"
  ansible.builtin.assert:
    that:
      - ansible_version.full is version('2.16.1', '>=')
    fail_msg: "chrome_win_cis needs ansible-core >= 2.16.1 (found {{ ansible_version.full }})."

- name: "PRELIM | AUDIT | Check the target is Windows"
  ansible.builtin.assert:
    that:
      - ansible_facts['os_family'] == 'Windows'
    fail_msg: "Unsupported target: {{ ansible_facts['distribution'] }}. chrome_win_cis supports Windows only."

- name: "PRELIM | AUDIT | Find chrome.exe"
  ansible.windows.win_stat:
    path: "{{ item }}"
    get_checksum: false
  loop: "{{ chrome_win_cis_exe_paths }}"
  register: discovered_chrome_win_cis_exe

- name: "PRELIM | AUDIT | Stop when Chrome is not installed and install is off"
  when:
    - chrome_win_cis_installed | length == 0
    - not chrome_win_cis_install
  block:
    - name: "PRELIM | AUDIT | Stop when Chrome is not installed and install is off | Report"
      ansible.builtin.debug:
        msg: "No Google Chrome on {{ inventory_hostname }} and chrome_win_cis_install is false: nothing to harden."

    - name: "PRELIM | AUDIT | Stop when Chrome is not installed and install is off | End this host"
      ansible.builtin.meta: end_host

- name: "PRELIM | AUDIT | Start the policy set"
  ansible.builtin.set_fact:
    chrome_win_cis_policies: {}
```

### 3.3 `tasks/install.yml`

What: installs the Enterprise MSI when Chrome is missing, then looks for `chrome.exe` again. Why: `win_package`
downloads and installs the MSI silently; `creates_path` makes a rerun do nothing. The second `win_stat` refreshes
`chrome_win_cis_installed` for the report (like `package_facts` after `dnf` on Linux).

```yaml
---
- name: "INSTALL | Google Chrome | Install the Enterprise MSI"
  when: chrome_win_cis_installed | length == 0
  ansible.windows.win_package:
    path: "{{ chrome_win_cis_msi_url }}"
    creates_path: "{{ chrome_win_cis_exe_paths[0] }}"
    state: present

- name: "INSTALL | Google Chrome | Find chrome.exe again"
  ansible.windows.win_stat:
    path: "{{ item }}"
    get_checksum: false
  loop: "{{ chrome_win_cis_exe_paths }}"
  register: discovered_chrome_win_cis_exe
```

### 3.4 Section files

Same as the Linux files, with `chrome_cis_` renamed. Shortcut instead of pasting each one:

```bash
cd ~/ansible-cis
for f in chrome_cis/tasks/section_*/*.yml; do
  sed 's/chrome_cis_/chrome_win_cis_/g' "$f" > "chrome_win_cis/${f#chrome_cis/}"
done
```

Then change by hand:

1. Tags of the 4 Windows-only rules: `not_applicable` → `remote_access` (2.8.2), `active_directory` (2.10.1),
   `attack_surface` (2.30), `privacy` (3.13).
2. `section_2/main.yml`: add 2.1 at the top.
3. `section_2/cis_2.1.x.yml`: new file (below).

The full files are listed below anyway.


### 3.5 `tasks/section_1/`

`tasks/section_1/main.yml`:

```yaml
---
- name: "SECTION 1 | 1.1 HTTP authentication"
  ansible.builtin.import_tasks:
    file: cis_1.1.x.yml

- name: "SECTION 1 | 1.2 Safe Browsing settings"
  ansible.builtin.import_tasks:
    file: cis_1.2.x.yml

- name: "SECTION 1 | 1.x Enforced Defaults"
  ansible.builtin.import_tasks:
    file: cis_1.x.yml
```

`tasks/section_1/cis_1.1.x.yml`:

```yaml
---
- name: "1.1.1 | PATCH | Ensure 'Cross-origin HTTP Authentication prompts' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_1_1
  tags:
    - level1
    - automated
    - patch
    - rule_1.1.1
    - http_auth
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AllowCrossOriginAuthPrompt': false}) }}"
```

`tasks/section_1/cis_1.2.x.yml`:

```yaml
---
- name: "1.2.1 | PATCH | Ensure 'Configure the list of domains on which Safe Browsing will not trigger warnings' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_2_1
  tags:
    - level1
    - automated
    - patch
    - rule_1.2.1
    - safe_browsing
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SafeBrowsingAllowlistDomains': []}) }}"

- name: "1.2.2 | PATCH | Ensure 'Safe Browsing Protection Level' is set to 'Enabled: Safe Browsing is active in the standard mode.' or higher"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_2_2
    - chrome_win_cis_safe_browsing_protection_level is not none
  tags:
    - level1
    - manual
    - patch
    - rule_1.2.2
    - safe_browsing
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SafeBrowsingProtectionLevel': chrome_win_cis_safe_browsing_protection_level}) }}"
```

`tasks/section_1/cis_1.x.yml`:

```yaml
---
- name: "1.3 | PATCH | Ensure 'Allow Google Cast to connect to Cast devices on all IP addresses' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_3
  tags:
    - level1
    - automated
    - patch
    - rule_1.3
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'MediaRouterCastAllowAllIPs': false}) }}"

- name: "1.4 | PATCH | Ensure 'Allow queries to a Google time service' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_4
  tags:
    - level1
    - automated
    - patch
    - rule_1.4
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'BrowserNetworkTimeQueriesEnabled': true}) }}"

- name: "1.5 | PATCH | Ensure 'Allow the audio sandbox to run' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_5
  tags:
    - level1
    - automated
    - patch
    - rule_1.5
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AudioSandboxEnabled': true}) }}"

- name: "1.6 | PATCH | Ensure 'Ask where to save each file before downloading' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_6
  tags:
    - level1
    - automated
    - patch
    - rule_1.6
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'PromptForDownloadLocation': true}) }}"

- name: "1.7 | PATCH | Ensure 'Continue running background apps when Google Chrome is closed' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_7
  tags:
    - level1
    - automated
    - patch
    - rule_1.7
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'BackgroundModeEnabled': false}) }}"

- name: "1.8 | PATCH | Ensure 'Control SafeSites adult content filtering' is set to 'Enabled: Filter top level sites (but not embedded iframes) for adult content'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_1_8
  tags:
    - level2
    - automated
    - patch
    - rule_1.8
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SafeSitesFilterBehavior': 1}) }}"

- name: "1.9 | PATCH | Ensure 'Determine the availability of variations' is set to 'Enable all variations'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_9
    - chrome_win_cis_variations is not none
  tags:
    - level1
    - manual
    - patch
    - rule_1.9
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ChromeVariations': chrome_win_cis_variations}) }}"

- name: "1.10 | PATCH | Ensure 'Disable Certificate Transparency enforcement for a list of Legacy Certificate Authorities' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_10
  tags:
    - level1
    - automated
    - patch
    - rule_1.10
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'CertificateTransparencyEnforcementDisabledForLegacyCas': []}) }}"

- name: "1.11 | PATCH | Ensure 'Disable Certificate Transparency enforcement for a list of subjectPublicKeyInfo hashes' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_11
  tags:
    - level1
    - automated
    - patch
    - rule_1.11
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'CertificateTransparencyEnforcementDisabledForCas': []}) }}"

- name: "1.12 | PATCH | Ensure 'Disable Certificate Transparency enforcement for a list of URLs' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_12
  tags:
    - level1
    - automated
    - patch
    - rule_1.12
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'CertificateTransparencyEnforcementDisabledForUrls': []}) }}"

- name: "1.13 | PATCH | Ensure 'Disable saving browser history' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_13
  tags:
    - level1
    - automated
    - patch
    - rule_1.13
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SavingBrowserHistoryDisabled': false}) }}"

- name: "1.14 | PATCH | Ensure 'DNS interception checks enabled' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_14
  tags:
    - level1
    - automated
    - patch
    - rule_1.14
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DNSInterceptionChecksEnabled': true}) }}"

- name: "1.15 | PATCH | Ensure 'Enable component updates in Google Chrome' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_15
  tags:
    - level1
    - automated
    - patch
    - rule_1.15
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ComponentUpdatesEnabled': true}) }}"

- name: "1.16 | PATCH | Ensure 'Enable globally scoped HTTP auth cache' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_16
  tags:
    - level1
    - automated
    - patch
    - rule_1.16
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'GloballyScopeHTTPAuthCacheEnabled': false}) }}"

- name: "1.17 | PATCH | Ensure 'Enable online OCSP/CRL checks' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_17
  tags:
    - level1
    - automated
    - patch
    - rule_1.17
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'EnableOnlineRevocationChecks': false}) }}"

- name: "1.18 | PATCH | Ensure 'Enable security warnings for command-line flags' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_18
  tags:
    - level1
    - automated
    - patch
    - rule_1.18
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'CommandLineFlagSecurityWarningsEnabled': true}) }}"

- name: "1.19 | PATCH | Ensure 'Enable third party software injection blocking' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_19
  tags:
    - level1
    - automated
    - patch
    - rule_1.19
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ThirdPartyBlockingEnabled': true}) }}"

- name: "1.20 | PATCH | Ensure 'Enables managed extensions to use the Enterprise Hardware Platform API' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_20
  tags:
    - level1
    - automated
    - patch
    - rule_1.20
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'EnterpriseHardwarePlatformAPIEnabled': false}) }}"

- name: "1.21 | PATCH | Ensure 'Ephemeral profile' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_21
  tags:
    - level1
    - automated
    - patch
    - rule_1.21
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ForceEphemeralProfiles': false}) }}"

- name: "1.22 | PATCH | Ensure 'Import autofill form data from default browser on first run' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_22
  tags:
    - level1
    - automated
    - patch
    - rule_1.22
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ImportAutofillFormData': false}) }}"

- name: "1.23 | PATCH | Ensure 'Import of homepage from default browser on first run' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_23
  tags:
    - level1
    - automated
    - patch
    - rule_1.23
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ImportHomepage': false}) }}"

- name: "1.24 | PATCH | Ensure 'Import search engines from default browser on first run' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_24
  tags:
    - level1
    - automated
    - patch
    - rule_1.24
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ImportSearchEngine': false}) }}"

- name: "1.25 | PATCH | Ensure 'List of names that will bypass the HSTS policy check' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_25
  tags:
    - level1
    - automated
    - patch
    - rule_1.25
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'HSTSPolicyBypassList': []}) }}"

- name: "1.26 | PATCH | Ensure 'Origins or hostname patterns for which restrictions on insecure origins should not apply' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_26
  tags:
    - level1
    - automated
    - patch
    - rule_1.26
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'OverrideSecurityRestrictionsOnInsecureOrigin': []}) }}"

- name: "1.27 | PATCH | Ensure 'Suppress lookalike domain warnings on domains' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_27
  tags:
    - level1
    - automated
    - patch
    - rule_1.27
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'LookalikeWarningAllowlistDomains': []}) }}"

- name: "1.28 | PATCH | Ensure 'Suppress the unsupported OS warning' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_28
  tags:
    - level1
    - automated
    - patch
    - rule_1.28
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SuppressUnsupportedOSWarning': false}) }}"

- name: "1.29 | PATCH | Ensure 'URLs for which local IPs are exposed in WebRTC ICE candidates' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_1_29
  tags:
    - level1
    - automated
    - patch
    - rule_1.29
    - enforced_defaults
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'WebRtcLocalIpsAllowedUrls': []}) }}"
```

### 3.6 `tasks/section_2/`

`tasks/section_2/main.yml`:

```yaml
---
- name: "SECTION 2 | 2.1 Google Update"
  ansible.builtin.import_tasks:
    file: cis_2.1.x.yml

- name: "SECTION 2 | 2.2 Content settings"
  ansible.builtin.import_tasks:
    file: cis_2.2.x.yml

- name: "SECTION 2 | 2.3 Extensions"
  ansible.builtin.import_tasks:
    file: cis_2.3.x.yml

- name: "SECTION 2 | 2.4 HTTP authentication"
  ansible.builtin.import_tasks:
    file: cis_2.4.x.yml

- name: "SECTION 2 | 2.5 Native Messaging"
  ansible.builtin.import_tasks:
    file: cis_2.5.x.yml

- name: "SECTION 2 | 2.6 Password manager"
  ansible.builtin.import_tasks:
    file: cis_2.6.x.yml

- name: "SECTION 2 | 2.7 Printing"
  ansible.builtin.import_tasks:
    file: cis_2.7.x.yml

- name: "SECTION 2 | 2.8 Remote access (Chrome Remote Desktop)"
  ansible.builtin.import_tasks:
    file: cis_2.8.x.yml

- name: "SECTION 2 | 2.9 First-Party Sets"
  ansible.builtin.import_tasks:
    file: cis_2.9.x.yml

- name: "SECTION 2 | 2.10 Microsoft Active Directory"
  ansible.builtin.import_tasks:
    file: cis_2.10.x.yml

- name: "SECTION 2 | 2.x Attack Surface Reduction"
  ansible.builtin.import_tasks:
    file: cis_2.x.yml
```

`tasks/section_2/cis_2.1.x.yml`:

New on Windows. Google Update settings live in their own key, so these two write directly with `win_regedit` instead of adding to `chrome_win_cis_policies`.

```yaml
---
- name: "2.1.1 | PATCH | Ensure 'Update policy override' is set to 'Enabled' with 'Always allow updates (recommended)' or 'Automatic silent updates' specified"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_1_1
  tags:
    - level1
    - automated
    - patch
    - rule_2.1.1
    - google_update
  ansible.windows.win_regedit:
    path: "{{ chrome_win_cis_update_key }}"
    name: "Update{8A69D345-D564-463C-AFF1-A69D9E530F96}"
    data: "{{ chrome_win_cis_update_policy }}"
    type: dword

- name: "2.1.2 | PATCH | Ensure 'Auto-update check period override' is set to any value except '0'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_1_2
  tags:
    - level1
    - automated
    - patch
    - rule_2.1.2
    - google_update
  ansible.windows.win_regedit:
    path: "{{ chrome_win_cis_update_key }}"
    name: AutoUpdateCheckPeriodMinutes
    data: "{{ chrome_win_cis_update_check_minutes }}"
    type: dword
```

`tasks/section_2/cis_2.2.x.yml`:

```yaml
---
- name: "2.2.1 | PATCH | Ensure 'Control use of insecure content exceptions' is set to 'Enabled: Do not allow any site to load mixed content'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_2_1
  tags:
    - level1
    - automated
    - patch
    - rule_2.2.1
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultInsecureContentSetting': 2}) }}"

- name: "2.2.2 | PATCH | Ensure 'Control use of the Web Bluetooth API' is set to 'Enabled: Do not allow any site to request access to Bluetooth devices via the Web Bluetooth API'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_2_2
  tags:
    - level2
    - automated
    - patch
    - rule_2.2.2
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultWebBluetoothGuardSetting': 2}) }}"

- name: "2.2.3 | PATCH | Ensure 'Control use of the WebUSB API' is set to 'Enabled: Do not allow any site to request access to USB devices via the WebUSB API'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_2_3
  tags:
    - level2
    - automated
    - patch
    - rule_2.2.3
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultWebUsbGuardSetting': 2}) }}"

- name: "2.2.4 | PATCH | Ensure 'Default notification setting' is set to 'Enabled: Do not allow any site to show desktop notifications'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_2_4
  tags:
    - level2
    - automated
    - patch
    - rule_2.2.4
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultNotificationsSetting': 2}) }}"

- name: "2.2.5 | PATCH | Ensure 'Allow local file access to file:// URLs on these sites in the PDF Viewer' Is Disabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_2_5
  tags:
    - level1
    - automated
    - patch
    - rule_2.2.5
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'PdfLocalFileAccessAllowedForDomains': []}) }}"
```

`tasks/section_2/cis_2.3.x.yml`:

```yaml
---
- name: "2.3.1 | PATCH | Ensure 'Blocks external extensions from being installed' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_3_1
  tags:
    - level1
    - automated
    - patch
    - rule_2.3.1
    - extensions
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'BlockExternalExtensions': true}) }}"

- name: "2.3.2 | PATCH | Ensure 'Configure allowed app/extension types' is set to 'Enabled: extension, hosted_app, platform_app, theme'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_3_2
  tags:
    - level1
    - automated
    - patch
    - rule_2.3.2
    - extensions
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ExtensionAllowedTypes': ['extension', 'hosted_app', 'platform_app', 'theme']}) }}"

- name: "2.3.3 | PATCH | Ensure 'Configure extension installation blocklist' is set to 'Enabled: *'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_3_3
  tags:
    - level1
    - automated
    - patch
    - rule_2.3.3
    - extensions
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ExtensionInstallBlocklist': ['*']}) }}"

- name: "2.3.3 | PATCH | Ensure 'Configure extension installation blocklist' is set to 'Enabled: *' | ExtensionInstallAllowlist"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_3_3
    - chrome_win_cis_extension_allowlist | length > 0
  tags:
    - level1
    - automated
    - patch
    - rule_2.3.3
    - extensions
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ExtensionInstallAllowlist': chrome_win_cis_extension_allowlist}) }}"

- name: "2.3.4 | PATCH | Ensure 'Default third-party storage partitioning setting' Is Enabled and Blocked"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_3_4
  tags:
    - level2
    - automated
    - patch
    - rule_2.3.4
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultThirdPartyStoragePartitioningSetting': 2}) }}"

- name: "2.3.5 | PATCH | Ensure 'Block third-party storage partitioning for these origins' Is Configured"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_3_5
    - chrome_win_cis_third_party_storage_blocked_origins | length > 0
  tags:
    - level1
    - manual
    - patch
    - rule_2.3.5
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ThirdPartyStoragePartitioningBlockedForOrigins': chrome_win_cis_third_party_storage_blocked_origins}) }}"

- name: "2.3.6 | PATCH | Ensure 'Control Manifest v2 extension availability' Is Set to Forced Only"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_3_6
  tags:
    - level2
    - automated
    - patch
    - rule_2.3.6
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ExtensionManifestV2Availability': 3}) }}"

- name: "2.3.7 | PATCH | Ensure 'Control availability of extensions unpublished on the Chrome Web Store' Is Disabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_3_7
  tags:
    - level1
    - automated
    - patch
    - rule_2.3.7
    - extensions
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ExtensionUnpublishedAvailability': 1}) }}"
```

`tasks/section_2/cis_2.4.x.yml`:

```yaml
---
- name: "2.4.1 | PATCH | Ensure 'Supported authentication schemes' is set to 'Enabled: ntlm, negotiate'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_4_1
  tags:
    - level2
    - automated
    - patch
    - rule_2.4.1
    - http_auth
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AuthSchemes': 'ntlm,negotiate'}) }}"
```

`tasks/section_2/cis_2.5.x.yml`:

```yaml
---
- name: "2.5.1 | PATCH | Ensure 'Configure native messaging blocklist' is set to 'Enabled: *'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_5_1
  tags:
    - level2
    - automated
    - patch
    - rule_2.5.1
    - native_messaging
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'NativeMessagingBlocklist': ['*']}) }}"

- name: "2.5.1 | PATCH | Ensure 'Configure native messaging blocklist' is set to 'Enabled: *' | NativeMessagingAllowlist"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_5_1
    - chrome_win_cis_native_messaging_allowlist | length > 0
  tags:
    - level2
    - automated
    - patch
    - rule_2.5.1
    - native_messaging
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'NativeMessagingAllowlist': chrome_win_cis_native_messaging_allowlist}) }}"
```

`tasks/section_2/cis_2.6.x.yml`:

```yaml
---
- name: "2.6.1 | PATCH | Ensure 'Enable saving passwords to the password manager' is Explicitly Configured"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_6_1
    - chrome_win_cis_password_manager_enabled is not none
  tags:
    - level1
    - manual
    - patch
    - rule_2.6.1
    - password_manager
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'PasswordManagerEnabled': chrome_win_cis_password_manager_enabled}) }}"
```

`tasks/section_2/cis_2.7.x.yml`:

```yaml
---
- name: "2.7.1 | PATCH | Ensure 'Enable Google Cloud Print Proxy' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_7_1
  tags:
    - level1
    - automated
    - patch
    - rule_2.7.1
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'CloudPrintProxyEnabled': false}) }}"
```

`tasks/section_2/cis_2.8.x.yml`:

```yaml
---
- name: "2.8.1 | PATCH | Ensure 'Allow remote access connections to this machine' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_8_1
    - chrome_win_cis_remote_access_connections is not none
  tags:
    - level1
    - manual
    - patch
    - rule_2.8.1
    - remote_access
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RemoteAccessHostAllowRemoteAccessConnections': chrome_win_cis_remote_access_connections}) }}"

- name: "2.8.2 | PATCH | Ensure 'Allow remote users to interact with elevated windows in remote assistance sessions' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_8_2
  tags:
    - level1
    - automated
    - patch
    - rule_2.8.2
    - remote_access
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RemoteAccessHostAllowUiAccessForRemoteAssistance': false}) }}"

- name: "2.8.3 | PATCH | Ensure 'Configure the required domain names for remote access clients' is set to 'Enabled'  with a domain defined"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_8_3
    - chrome_win_cis_remote_access_client_domains | length > 0
  tags:
    - level1
    - manual
    - patch
    - rule_2.8.3
    - remote_access
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RemoteAccessHostClientDomainList': chrome_win_cis_remote_access_client_domains}) }}"

- name: "2.8.4 | PATCH | Ensure 'Enable curtaining of remote access hosts' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_8_4
  tags:
    - level1
    - automated
    - patch
    - rule_2.8.4
    - remote_access
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RemoteAccessHostRequireCurtain': false}) }}"

- name: "2.8.5 | PATCH | Ensure 'Enable firewall traversal from remote access host' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_8_5
  tags:
    - level1
    - automated
    - patch
    - rule_2.8.5
    - remote_access
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RemoteAccessHostFirewallTraversal': false}) }}"

- name: "2.8.6 | PATCH | Ensure 'Enable or disable PIN-less authentication for remote access hosts' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_8_6
  tags:
    - level1
    - automated
    - patch
    - rule_2.8.6
    - remote_access
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RemoteAccessHostAllowClientPairing': false}) }}"

- name: "2.8.7 | PATCH | Ensure 'Enable the use of relay servers by the remote access host' is set to 'Disabled'."
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_8_7
  tags:
    - level1
    - automated
    - patch
    - rule_2.8.7
    - remote_access
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RemoteAccessHostAllowRelayedConnection': false}) }}"
```

`tasks/section_2/cis_2.9.x.yml`:

```yaml
---
- name: "2.9.1 | PATCH | Ensure 'Enable First-Party Sets' Is Disabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_9_1
  tags:
    - level1
    - manual
    - patch
    - rule_2.9.1
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'FirstPartySetsEnabled': false}) }}"
```

`tasks/section_2/cis_2.10.x.yml`:

```yaml
---
- name: "2.10.1 | PATCH | Ensure 'Allow automatic sign-in to Microsoft cloud identity providers' Is Enabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_10_1
  tags:
    - level1
    - manual
    - patch
    - rule_2.10.1
    - active_directory
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'CloudAPAuthEnabled': true}) }}"
```

`tasks/section_2/cis_2.x.yml`:

```yaml
---
- name: "2.11 | PATCH | Ensure 'Allow download restrictions' is set to 'Enabled: Block malicious downloads'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_11
  tags:
    - level1
    - automated
    - patch
    - rule_2.11
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DownloadRestrictions': 4}) }}"

- name: "2.12 | PATCH | Ensure 'Allow proceeding from the SSL warning page' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_12
  tags:
    - level2
    - automated
    - patch
    - rule_2.12
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SSLErrorOverrideAllowed': false}) }}"

- name: "2.13 | PATCH | Ensure 'Disable proceeding from the Safe Browsing warning page' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_13
  tags:
    - level1
    - automated
    - patch
    - rule_2.13
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DisableSafeBrowsingProceedAnyway': true}) }}"

- name: "2.14 | PATCH | Ensure 'Require Site Isolation for every site' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_14
  tags:
    - level1
    - automated
    - patch
    - rule_2.14
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SitePerProcess': true}) }}"

- name: "2.15 | PATCH | Ensure 'Force Google SafeSearch' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_15
  tags:
    - level2
    - automated
    - patch
    - rule_2.15
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ForceGoogleSafeSearch': true}) }}"

- name: "2.16 | PATCH | Ensure 'Notify a user that a browser relaunch or device restart is recommended or required' is set to 'Enabled: Show a recurring prompt to the user indication that a relaunch is required'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_16
  tags:
    - level1
    - automated
    - patch
    - rule_2.16
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RelaunchNotification': 2}) }}"

- name: "2.17 | PATCH | Ensure 'Proxy settings' is set to 'Enabled' and does not contain \"ProxyMode\": \"auto_detect\""
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_17
    - chrome_win_cis_proxy_mode is not none
  tags:
    - level1
    - automated
    - patch
    - rule_2.17
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ProxyMode': chrome_win_cis_proxy_mode}) }}"

- name: "2.18 | PATCH | Ensure 'Require online OCSP/CRL checks for local trust anchors' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_18
  tags:
    - level2
    - automated
    - patch
    - rule_2.18
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RequireOnlineRevocationChecksForLocalAnchors': true}) }}"

- name: "2.19 | PATCH | Ensure 'Set the time period for update notifications' is set to 'Enabled: 86400000'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_19
  tags:
    - level1
    - automated
    - patch
    - rule_2.19
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RelaunchNotificationPeriod': 86400000}) }}"

- name: "2.20 | PATCH | Ensure 'Allow Web Authentication requests on sites with broken TLS certificates' Is Disabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_20
  tags:
    - level1
    - automated
    - patch
    - rule_2.20
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AllowWebAuthnWithBrokenTlsCerts': false}) }}"

- name: "2.21 | PATCH | Ensure 'Allow reporting of domain reliability related data' Is Disabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_21
  tags:
    - level1
    - automated
    - patch
    - rule_2.21
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DomainReliabilityAllowed': false}) }}"

- name: "2.22 | PATCH | Ensure 'Enable TLS Encrypted ClientHello' Is Enabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_22
  tags:
    - level1
    - automated
    - patch
    - rule_2.22
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'EncryptedClientHelloEnabled': true}) }}"

- name: "2.23 | PATCH | Ensure 'Determines whether the built-in certificate verifier will enforce constraints encoded into trust anchors loaded from the platform trust store' Is Enabled"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_2_23
  tags:
    - level2
    - automated
    - patch
    - rule_2.23
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'EnforceLocalAnchorConstraintsEnabled': true}) }}"

- name: "2.24 | PATCH | Ensure 'Keep browsing data when creating enterprise profile by default' Is Enabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_24
  tags:
    - level1
    - automated
    - patch
    - rule_2.24
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'EnterpriseProfileCreationKeepBrowsingData': true}) }}"

- name: "2.25 | PATCH | Ensure 'Allow file or directory picker APIs to be called without prior user gesture' Is Disabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_25
  tags:
    - level1
    - automated
    - patch
    - rule_2.25
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'FileOrDirectoryPickerWithoutGestureAllowedForOrigins': []}) }}"

- name: "2.26 | PATCH | Ensure 'Enable Google Search Side Panel' Is Disabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_26
  tags:
    - level1
    - automated
    - patch
    - rule_2.26
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'GoogleSearchSidePanelEnabled': false}) }}"

- name: "2.27 | PATCH | Ensure 'Http Allowlist' Is Properly Configured"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_27
    - chrome_win_cis_http_allowlist | length > 0
  tags:
    - level1
    - manual
    - patch
    - rule_2.27
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'HttpAllowlist': chrome_win_cis_http_allowlist}) }}"

- name: "2.28 | PATCH | Ensure 'Enable automatic HTTPS upgrades' Is Enabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_28
  tags:
    - level1
    - automated
    - patch
    - rule_2.28
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'HttpsUpgradesEnabled': true}) }}"

- name: "2.29 | PATCH | Ensure 'Insecure Hashes in TLS Handshakes Enabled' Is Disabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_29
  tags:
    - level1
    - automated
    - patch
    - rule_2.29
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'InsecureHashesInTLSHandshakesEnabled': false}) }}"

- name: "2.30 | PATCH | Ensure 'Enable Renderer App Container' Is Enabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_30
  tags:
    - level1
    - automated
    - patch
    - rule_2.30
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RendererAppContainerEnabled': true}) }}"

- name: "2.31 | PATCH | Ensure 'Enable strict MIME type checking for worker scripts' Is Enabled"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_31
  tags:
    - level1
    - automated
    - patch
    - rule_2.31
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'StrictMimetypeCheckForWorkerScriptsEnabled': true}) }}"

- name: "2.32 | PATCH | Ensure 'Allow remote debugging' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_2_32
  tags:
    - level1
    - automated
    - patch
    - rule_2.32
    - attack_surface
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'RemoteDebuggingAllowed': false}) }}"
```

### 3.7 `tasks/section_3/`

`tasks/section_3/main.yml`:

```yaml
---
- name: "SECTION 3 | 3.1 Content settings"
  ansible.builtin.import_tasks:
    file: cis_3.1.x.yml

- name: "SECTION 3 | 3.2 Google Cast"
  ansible.builtin.import_tasks:
    file: cis_3.2.x.yml

- name: "SECTION 3 | 3.x Privacy"
  ansible.builtin.import_tasks:
    file: cis_3.x.yml
```

`tasks/section_3/cis_3.1.x.yml`:

```yaml
---
- name: "3.1.1 | PATCH | Ensure 'Default cookies setting' is set to 'Enabled: Keep cookies for the duration of the session'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_3_1_1
  tags:
    - level2
    - automated
    - patch
    - rule_3.1.1
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultCookiesSetting': 4}) }}"

- name: "3.1.2 | PATCH | Ensure 'Default geolocation setting' is set to 'Enabled: Do not allow any site to track the users' physical location'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_1_2
  tags:
    - level1
    - automated
    - patch
    - rule_3.1.2
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultGeolocationSetting': 2}) }}"
```

`tasks/section_3/cis_3.2.x.yml`:

```yaml
---
- name: "3.2.1 | PATCH | Ensure 'Enable Google Cast' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_2_1
  tags:
    - level1
    - automated
    - patch
    - rule_3.2.1
    - google_cast
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'EnableMediaRouter': false}) }}"
```

`tasks/section_3/cis_3.x.yml`:

```yaml
---
- name: "3.3 | PATCH | Ensure 'Allow websites to query for available payment methods' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_3
  tags:
    - level1
    - automated
    - patch
    - rule_3.3
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'PaymentMethodQueryEnabled': false}) }}"

- name: "3.4 | PATCH | Ensure 'Block third party cookies' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_4
  tags:
    - level1
    - automated
    - patch
    - rule_3.4
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'BlockThirdPartyCookies': true}) }}"

- name: "3.5 | PATCH | Ensure 'Browser sign in settings' is set to 'Enabled: Disabled browser sign-in'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_3_5
  tags:
    - level2
    - automated
    - patch
    - rule_3.5
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'BrowserSignin': 0}) }}"

- name: "3.6 | PATCH | Ensure 'Control how Chrome Cleanup reports data to Google' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_6
  tags:
    - level1
    - automated
    - patch
    - rule_3.6
    - not_applicable
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ChromeCleanupReportingEnabled': false}) }}"

- name: "3.7 | PATCH | Ensure 'Disable synchronization of data with Google' is set to 'Enabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_7
  tags:
    - level1
    - automated
    - patch
    - rule_3.7
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SyncDisabled': true}) }}"

- name: "3.8 | PATCH | Ensure 'Enable alternate error pages' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_8
  tags:
    - level1
    - automated
    - patch
    - rule_3.8
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AlternateErrorPagesEnabled': false}) }}"

- name: "3.9 | PATCH | Ensure 'Enable deleting browser and download history' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_9
  tags:
    - level1
    - automated
    - patch
    - rule_3.9
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AllowDeletingBrowserHistory': false}) }}"

- name: "3.10 | PATCH | Ensure 'Enable predict network actions` is set to 'Enabled: Do not predict actions on any network connection'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_10
  tags:
    - level1
    - automated
    - patch
    - rule_3.10
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'NetworkPredictionOptions': 2}) }}"

- name: "3.11 | PATCH | Ensure 'Enable or disable spell checking web service' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_11
  tags:
    - level1
    - automated
    - patch
    - rule_3.11
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SpellCheckServiceEnabled': false}) }}"

- name: "3.12 | PATCH | Ensure 'Enable reporting of usage and crash-related data' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_12
  tags:
    - level1
    - automated
    - patch
    - rule_3.12
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'MetricsReportingEnabled': false}) }}"

- name: "3.13 | PATCH | Ensure 'Enable Safe Browsing for trusted sources' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_13
  tags:
    - level1
    - automated
    - patch
    - rule_3.13
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SafeBrowsingForTrustedSourcesEnabled': false}) }}"

- name: "3.14 | PATCH | Ensure 'Enable search suggestions' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_3_14
  tags:
    - level2
    - automated
    - patch
    - rule_3.14
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SearchSuggestEnabled': false}) }}"

- name: "3.15 | PATCH | Ensure 'Enable Translate' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_3_15
  tags:
    - level2
    - automated
    - patch
    - rule_3.15
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'TranslateEnabled': false}) }}"

- name: "3.16 | PATCH | Ensure 'Enable URL-keyed anonymized data collection' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_3_16
  tags:
    - level1
    - automated
    - patch
    - rule_3.16
    - privacy
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'UrlKeyedAnonymizedDataCollectionEnabled': false}) }}"
```

### 3.8 `tasks/section_4/`

`tasks/section_4/main.yml`:

```yaml
---
- name: "SECTION 4 | 4.1 Allow or deny screen capture"
  ansible.builtin.import_tasks:
    file: cis_4.1.x.yml

- name: "SECTION 4 | 4.2 Content settings"
  ansible.builtin.import_tasks:
    file: cis_4.2.x.yml

- name: "SECTION 4 | 4.x Data Loss Prevention"
  ansible.builtin.import_tasks:
    file: cis_4.x.yml
```

`tasks/section_4/cis_4.1.x.yml`:

```yaml
---
- name: "4.1.1 | PATCH | Ensure 'Allow or deny screen capture' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_1_1
  tags:
    - level2
    - automated
    - patch
    - rule_4.1.1
    - screen_capture
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ScreenCaptureAllowed': false}) }}"
```

`tasks/section_4/cis_4.2.x.yml`:

```yaml
---
- name: "4.2.1 | PATCH | Ensure 'Control use of the Serial API' is set to 'Enabled: Do not allow any site to request access to serial ports via the Serial API'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_2_1
  tags:
    - level2
    - automated
    - patch
    - rule_4.2.1
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultSerialGuardSetting': 2}) }}"

- name: "4.2.2 | PATCH | Ensure 'Default Sensors Setting' is set to 'Enabled: Do not allow any site to access sensors'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_2_2
  tags:
    - level2
    - automated
    - patch
    - rule_4.2.2
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultSensorsSetting': 2}) }}"

- name: "4.2.3 | PATCH | Ensure 'Allow clipboard for these sites' Is Configured"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_4_2_3
    - chrome_win_cis_clipboard_allowed_urls | length > 0
  tags:
    - level1
    - manual
    - patch
    - rule_4.2.3
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ClipboardAllowedForUrls': chrome_win_cis_clipboard_allowed_urls}) }}"

- name: "4.2.4 | PATCH | Ensure 'Block clipboard on these sites' Is Configured"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_4_2_4
    - chrome_win_cis_clipboard_blocked_urls | length > 0
  tags:
    - level1
    - manual
    - patch
    - rule_4.2.4
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ClipboardBlockedForUrls': chrome_win_cis_clipboard_blocked_urls}) }}"

- name: "4.2.5 | PATCH | Ensure 'Default clipboard setting' Is 'Enabled' to 'Deny Permissions'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_4_2_5
  tags:
    - level1
    - automated
    - patch
    - rule_4.2.5
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultClipboardSetting': 2}) }}"

- name: "4.2.6 | PATCH | Ensure 'Default Window Management permissions setting' Is 'Enabled' to 'Deny Permission'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_2_6
  tags:
    - level2
    - automated
    - patch
    - rule_4.2.6
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DefaultWindowManagementSetting': 2}) }}"

- name: "4.2.7 | PATCH | Ensure 'Allow Window Management permission on these sites' Is Configured"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_2_7
    - chrome_win_cis_window_management_allowed_urls | length > 0
  tags:
    - level2
    - manual
    - patch
    - rule_4.2.7
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'WindowManagementAllowedForUrls': chrome_win_cis_window_management_allowed_urls}) }}"

- name: "4.2.8 | PATCH | Ensure 'Block Window Management permission on these sites' Is Configured"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_2_8
    - chrome_win_cis_window_management_blocked_urls | length > 0
  tags:
    - level2
    - manual
    - patch
    - rule_4.2.8
    - content_settings
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'WindowManagementBlockedForUrls': chrome_win_cis_window_management_blocked_urls}) }}"
```

`tasks/section_4/cis_4.x.yml`:

```yaml
---
- name: "4.3 | PATCH | Ensure 'Allow invocation of file selection dialogs' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_3
  tags:
    - level2
    - automated
    - patch
    - rule_4.3
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AllowFileSelectionDialogs': false}) }}"

- name: "4.4 | PATCH | Ensure 'Allow or deny audio capture' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_4
  tags:
    - level2
    - automated
    - patch
    - rule_4.4
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AudioCaptureAllowed': false}) }}"

- name: "4.5 | PATCH | Ensure 'Allow or deny video capture' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_5
  tags:
    - level2
    - automated
    - patch
    - rule_4.5
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'VideoCaptureAllowed': false}) }}"

- name: "4.6 | PATCH | Ensure 'Allow user feedback' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_4_6
  tags:
    - level1
    - automated
    - patch
    - rule_4.6
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'UserFeedbackAllowed': false}) }}"

- name: "4.7 | PATCH | Ensure 'Controls the mode of DNS-over-HTTPS' is set to 'Enabled: DNS-over-HTTPS without insecure fallback'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_7
  tags:
    - level2
    - automated
    - patch
    - rule_4.7
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DnsOverHttpsMode': 'secure'}) }}"

- name: "4.8 | PATCH | Ensure 'Enable AutoFill for addresses' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_8
  tags:
    - level2
    - automated
    - patch
    - rule_4.8
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AutofillAddressEnabled': false}) }}"

- name: "4.9 | PATCH | Ensure 'Enable AutoFill for credit cards' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_4_9
  tags:
    - level1
    - automated
    - patch
    - rule_4.9
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'AutofillCreditCardEnabled': false}) }}"

- name: "4.10 | PATCH | Ensure 'Import saved passwords from default browser on first run' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_4_10
  tags:
    - level1
    - automated
    - patch
    - rule_4.10
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ImportSavedPasswords': false}) }}"

- name: "4.11 | PATCH | Ensure 'List of types that should be excluded from synchronization' is set to 'Enabled: passwords'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_4_11
  tags:
    - level1
    - automated
    - patch
    - rule_4.11
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'SyncTypesListDisabled': ['passwords']}) }}"

- name: "4.12 | PATCH | Ensure 'Allow or deny screen capture' is set to 'Disabled'"
  # same policy as 4.1.1, follows chrome_win_cis_rule_4_1_1
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_4_1_1
  tags:
    - level2
    - automated
    - patch
    - rule_4.12
    - data_loss_prevention
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'ScreenCaptureAllowed': false}) }}"
```

### 3.9 `tasks/section_5/`

`tasks/section_5/main.yml`:

```yaml
---
- name: "SECTION 5 | 5.x Forensics"
  ansible.builtin.import_tasks:
    file: cis_5.x.yml
```

`tasks/section_5/cis_5.x.yml`:

```yaml
---
- name: "5.1 | PATCH | Ensure 'Enable guest mode in browser' is set to 'Disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_5_1
  tags:
    - level2
    - automated
    - patch
    - rule_5.1
    - forensics
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'BrowserGuestModeEnabled': false}) }}"

- name: "5.2 | PATCH | Ensure 'Incognito mode availability' is set to 'Enabled: Incognito mode disabled'"
  when:
    - chrome_win_cis_level_2
    - chrome_win_cis_rule_5_2
  tags:
    - level2
    - automated
    - patch
    - rule_5.2
    - forensics
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'IncognitoModeAvailability': 1}) }}"

- name: "5.3 | PATCH | Ensure 'Set disk cache size, in bytes' is set to 'Enabled: 250609664'"
  when:
    - chrome_win_cis_level_1
    - chrome_win_cis_rule_5_3
  tags:
    - level1
    - automated
    - patch
    - rule_5.3
    - forensics
  ansible.builtin.set_fact:
    chrome_win_cis_policies: "{{ chrome_win_cis_policies | combine({'DiskCacheSize': 250609664}) }}"
```

### 3.10 `tasks/post.yml`

What: writes every collected policy, then reports. Why: the Windows version of Linux's `copy` task.

| Task | Does |
|---|---|
| Write the single-value policies | one `win_regedit` per policy: true/false become 1/0 (`REG_DWORD`), numbers `REG_DWORD`, text `REG_SZ` |
| Write the list policies | runs `post_list.yml` once per list policy; `apply: tags: always` so `--tags` runs still write them |
| Report | same as Linux |

```yaml
---
- name: "POST | PATCH | Write the single-value policies"
  ansible.windows.win_regedit:
    path: "{{ chrome_win_cis_policy_key }}"
    name: "{{ item.key }}"
    data: "{{ item.value | int if item.value is boolean else item.value }}"
    type: "{{ 'string' if item.value is string else 'dword' }}"
  loop: "{{ chrome_win_cis_value_policies }}"
  loop_control:
    label: "{{ item.key }}"

- name: "POST | PATCH | Write the list policies"
  ansible.builtin.include_tasks:
    file: post_list.yml
    apply:
      tags:
        - always
  loop: "{{ chrome_win_cis_list_policies }}"
  loop_control:
    loop_var: chrome_win_cis_list
    label: "{{ chrome_win_cis_list.key }}"

- name: "POST | AUDIT | Report"
  ansible.builtin.debug:
    msg:
      benchmark: "{{ chrome_win_cis_benchmark }}"
      chrome: "{{ chrome_win_cis_installed }}"
      policy_key: "{{ chrome_win_cis_policy_key }}"
      policies_written: "{{ chrome_win_cis_policies | length }}"
      levels: "Level 1 {{ 'on' if chrome_win_cis_level_1 else 'off' }}, Level 2 {{ 'on' if chrome_win_cis_level_2 else 'off' }}"
```

### 3.11 `tasks/post_list.yml`

What: writes one list policy. Why: Group Policy stores a list as a subkey with values `1`, `2`, `3` ...

| Task | Does |
|---|---|
| Read the current items | `win_reg_stat` lists the value names already in the subkey (read only) |
| Write the items | `ansible_loop.index` gives 1, 2, 3 ... as the value names |
| Remove items no longer in the list | deletes every value name that is not 1..n, e.g. `3` when the list shrank to 2 |

```yaml
---
# One list policy: chrome_win_cis_list.key = policy name, chrome_win_cis_list.value = its items.
# Stored like Group Policy does it: subkey <policy> with values 1, 2, 3 ... (empty list = no values).
- name: "POST | AUDIT | Read the current items of {{ chrome_win_cis_list.key }}"
  ansible.windows.win_reg_stat:
    path: "{{ chrome_win_cis_policy_key }}\\{{ chrome_win_cis_list.key }}"
  register: discovered_chrome_win_cis_list

- name: "POST | PATCH | Write the items of {{ chrome_win_cis_list.key }}"
  ansible.windows.win_regedit:
    path: "{{ chrome_win_cis_policy_key }}\\{{ chrome_win_cis_list.key }}"
    name: "{{ ansible_loop.index }}"
    data: "{{ item }}"
    type: string
  loop: "{{ chrome_win_cis_list.value }}"
  loop_control:
    extended: true

- name: "POST | PATCH | Remove items no longer in {{ chrome_win_cis_list.key }}"
  ansible.windows.win_regedit:
    path: "{{ chrome_win_cis_policy_key }}\\{{ chrome_win_cis_list.key }}"
    name: "{{ item }}"
    state: absent
  loop: "{{ discovered_chrome_win_cis_list.properties | default({}) | list | difference(range(1, chrome_win_cis_list.value | length + 1) | map('string') | list) }}"
```

## 4. Check your pasting (no VM needed)

```bash
cd ~/ansible-cis/chrome_win_cis
yamllint .
ansible-lint
```

## 5. Run on the Windows 11 VM

```bash
ansible-playbook -i inventory.ini playbooks/site.yml -k --check --diff   # dry run
ansible-playbook -i inventory.ini playbooks/site.yml -k                  # failed=0
ansible-playbook -i inventory.ini playbooks/site.yml -k                  # rerun: changed=0
```

On the VM:

```powershell
reg query HKLM\SOFTWARE\Policies\Google\Chrome
reg query HKLM\SOFTWARE\Policies\Google\Chrome\ExtensionInstallBlocklist
reg query HKLM\SOFTWARE\Policies\Google\Update
```

In Chrome: `chrome://policy` → **Reload policies**: Source = Platform, Applies to = Machine, Level = Mandatory.

Undo: restore the `pre-chrome` snapshot, or

```powershell
reg delete HKLM\SOFTWARE\Policies\Google\Chrome /f
reg delete HKLM\SOFTWARE\Policies\Google\Update /f
```
