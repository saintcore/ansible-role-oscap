saintcore.oscap
=========

[![lint](https://github.com/saintcore/ansible-role-oscap/actions/workflows/lint.yml/badge.svg)](https://github.com/saintcore/ansible-role-oscap/actions/workflows/lint.yml)
[![Molecule](https://github.com/saintcore/ansible-role-oscap/actions/workflows/molecule.yml/badge.svg)](https://github.com/saintcore/ansible-role-oscap/actions/workflows/molecule.yml)

This role performs OpenSCAP compliance and vulnerability scans on Linux systems.

It ensures OpenSCAP tools are installed and then executes scans based on boolean toggles. The role has two main functions:

* **Compliance Scan (`oscap_compliance_scan: true`)**
    * Runs `oscap xccdf eval` against a security profile (e.g., CIS).
    * Can use OS-specific defaults (EL9, Debian 12) or custom-provided SCAP content.

* **Vulnerability Scan (`oscap_oval_scan: true`)**
    * Runs `oscap oval eval` using OVAL (Open Vulnerability and Assessment Language) definitions.
    * Requires a URL to the OVAL definition file.

Both scan types can be run, and the role will optionally fetch the generated HTML reports back to the Ansible control node for review.

> **Note:** While the role includes defaults for **EL9 (RHEL, Rocky)** it can be used on **any** operating system by manually providing the required variables.

Requirements
------------

* This role **requires privilege escalation** (e.g., `become: true`) to install packages and execute the `oscap` scan command.
* The OVAL scan functionality requires the **`bzip2`** package on the target node to decompress the definitions.
* If `oscap_reports_store` is set to `true`, the user running `ansible-playbook` must have **write permissions** to the specified `oscap_reports_store_path` on the Ansible control node.

Role Variables
--------------

### Main Toggles & Report Settings

These are the primary controls for the role, defined in `defaults/main.yml`.

| Variable | Description | Default |
| :--- | :--- | :--- |
| **`oscap_compliance_scan`** | If `true`, run the `xccdf` compliance scan. | `true` |
| **`oscap_oval_scan`** | If `true`, run the `oval` vulnerability scan. | `false` |
| **`oscap_reports_store`** | If `true`, fetches the HTML reports from the target back to the Ansible controller. | `false` |
| **`oscap_reports_store_path`** | Destination path on the **controller** for fetched reports. | 'files/scap-results/{{ ansible_date_time.iso82_basic_short }}-{{ inventory_hostname }}/' |
| **`oscap_cleanup`** | If `true`, deletes all temporary files (reports and oval-content) from the target node after the scan. | `true` |

---

### Compliance Scan Variables (`oscap_compliance_scan: true`)

These variables control the `xccdf` compliance scan and are used to bypass OS-default logic.

| Variable | Description | Default |
| :--- | :--- | :--- |
| **`oscap_package`** | Manually specify the scanner package name. This bypasses OS-specific logic and is useful for unsupported systems. | `''` |
| **`oscap_content`** | Path to a **local** SCAP datastream file (on the controller) to be copied to the target. Required for unsupported systems. | `''` |
| **`oscap_tailoring`** | Path to a **local** XCCDF tailoring file (on the controller) to be copied to the target. | `''` |
| **`oscap_profile`** | The specific profile ID to evaluate. Required for unsupported systems. | `''` |

---

### OVAL Scan (`oscap_oval_scan: true`)

This functionality is controlled by the `oscap_oval_scan` boolean. To run an OVAL scan, you must set it to `true`.

The scan requires the **`oscap_oval_def_url`** variable, which points to the compressed OVAL definitions (`.xml.bz2`).

* This URL is **provided by default** for supported systems like **Red Hat 9** (defined in `vars/RedHat-9.yml`).
* For **unsupported systems** (or systems without a default, like Rocky 9), you must manually set `oscap_oval_def_url` in your playbook, group, or host variables for the scan to work.
* **Note for Rocky 9:** The role sets `oscap_oval_scan: false` by default in `vars/Rocky-9.yml`, as the official Rocky 9 OVAL data is stale.
---

### Internal OS-Specific Variables

These variables are loaded automatically from the `vars/` directory based on the target OS. You should not need to override these directly, as they provide the defaults for supported systems.

| Variable | Description |
| :--- | :--- |
| **`oscap_scanner_package`** | The default package name for the OpenSCAP scanner (e.g., `openscap-scanner`). |
| **`oscap_guide_package`** | The default package for the SCAP Security Guide (e.g., `scap-security-guide`). |
| **`oscap_guide_default_content`** | The absolute path to the default SCAP datastream on the target OS. |
| **`oscap_guide_default_profile`** | The default compliance profile used if `oscap_profile` is not set. |
| **`oscap_oval_def_url`** | The OS-specific URL for OVAL definitions (used when `oscap_oval_scan` is true). |

Dependencies
------------

This role has no external dependencies on other Ansible Galaxy roles.

Example Playbook
----------------

### Example 1: Default Compliance Scan on Rocky Linux 9

This example runs the role on a group of Rocky Linux 9 servers. It uses all the built-in defaults for that OS (compliance scan, CIS profile) and simply sets `oscap_reports_store` to `true` to fetch the results.

```yaml
- hosts: rockylinux_servers
  become: true
  roles:
    - { role: saintcore.oscap, oscap_reports_store: true }
```

### Example 2: OVAL Vulnerability Scan on Debian 12

This example runs only the OVAL scan on Debian 12 servers. We disable the compliance scan, enable the OVAL scan, and provide the required URL (which could also be set in host_vars/debian-12.yml).

```yaml
- hosts: debian_12_servers
  become: true
  roles:
    - role: saintcore.oscap
      # --- Main Toggles ---
      oscap_compliance_scan: false  # Disable compliance scan
      oscap_oval_scan: true         # Enable OVAL scan
      oscap_reports_store: true     # Fetch the HTML report

      # --- OVAL Variable ---
      oscap_oval_def_url: "[https://www.debian.org/security/oval/oval-definitions-bookworm.xml.bz2](https://www.debian.org/security/oval/oval-definitions-bookworm.xml.bz2)"
```


### Example 3: Custom Compliance Scan on Unsupported OS (Ubuntu 24.04)

This example shows how to use the role on an OS that doesn't have built-in variables (like Ubuntu). We manually provide the package name, the path to our own SCAP content, and the profile to use.

This assumes you have your custom ssg-ubuntu2404-ds.xml and ubuntu-tailoring.xml files in a files/ directory alongside your playbook.

```yaml
- hosts: ubuntu_servers
  become: true
  roles:
    - role: saintcore.oscap
      # --- Manual Overrides for Ubuntu ---
      oscap_package: "openscap-scanner"
      oscap_content: "files/ssg-ubuntu2404-ds.xml"
      oscap_tailoring: "files/ubuntu-tailoring.xml"
      oscap_profile: "xccdf_org.ssgproject.content_profile_cis_l1_server"

      # --- Report Fetching ---
      oscap_reports_store: true
      oscap_reports_store_path: "files/scap-results/ubuntu/{{ inventory_hostname }}/"
```

License
-------

* This role is licensed under **AGPL-3.0-only**.
* This identifier is defined by the SPDX project. You can view the specific license details here: [AGPL-3.0-only on SPDX](https://spdx.org/licenses/AGPL-3.0-only.html)

Author Information
------------------

This role was created by **saintcore**. You can find more of my work on GitHub: [github.com/saintcore](https://github.com/saintcore)