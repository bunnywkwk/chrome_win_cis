# chrome_win_cis

Ansible role that hardens Google Chrome on Windows 10/11 to the **CIS Google Chrome Benchmark v3.0.0** by writing
the policy registry values the benchmark names (`HKLM:\SOFTWARE\Policies\Google\Chrome`, Google Update under
`...\Google\Update`). Sibling of `chrome_cis` (RHEL): same rule IDs, toggles and levels.

Status: **not tested yet**. The tasks are typed from `docs/build-guide.md`.

- Ansible: ansible-core >= 2.16.1, `pywinrm` on the control node
- Collections: `ansible-galaxy collection install -r requirements.yml` (`ansible.windows`)
- Target: Windows with WinRM enabled (build guide section 2)

## Quick start

```yaml
- hosts: windows
  roles:
    - role: chrome_win_cis
      vars:
        chrome_win_cis_install: true
```

## Tailoring

All settings are in `defaults/main.yml`; override them in `group_vars`:

```yaml
chrome_win_cis_level_2: true        # Level 1 + Level 2
chrome_win_cis_rule_4_7: false      # skip one rule
chrome_win_cis_extension_allowlist: ["aapbdbdomjkkjkaonfhkkikfgjllcleb"]
```

Run one rule: `--tags rule_2.3.3`. Skip one: `--skip-tags rule_4.7`. Dry run: `--check --diff`.

A rule switched off later keeps its registry value; delete it with `reg delete` or restore a snapshot.
Rules marked `# WARNING:` in `defaults/main.yml` can break things; turn them off in `group_vars` if needed.
