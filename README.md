saintcore.oscap
=========

[![lint](https://github.com/saintcore/ansible-role-oscap/actions/workflows/lint.yml/badge.svg)](https://github.com/saintcore/ansible-role-oscap/actions/workflows/lint.yml)
[![Molecule](https://github.com/saintcore/ansible-role-oscap/actions/workflows/molecule.yml/badge.svg)](https://github.com/saintcore/ansible-role-oscap/actions/workflows/molecule.yml)

This role performs OpenSCAP compliance scans on **EL9 (RHEL 9, Rocky 9)** systems.

It ensures OpenSCAP tools are installed and then executes an `oscap xccdf eval` scan. The role can be configured to:

* Use the default SCAP Security Guide (SSG) profile for RHEL 9 or Rocky 9.
* Use custom-provided SCAP datastreams (`oscap_content`) and tailoring files (`oscap_tailoring`).
* Optionally fetch the generated HTML and XML-ARF reports back to the Ansible control node for review.

> **Note:** While the role includes defaults for EL9, it can be used on **any** operating system by manually providing the following variables: `oscap_package`, `oscap_content`, and `oscap_profile`. Doing so will bypass all OS-specific default logic.

Requirements
------------

* This role **requires privilege escalation** (e.g., `become: true`) to install packages and execute the `oscap` scan command.
* The role's default variables are designed for **EL9 (RHEL 9, Rocky 9)**.
* If `oscap_reports_store` is set to `true`, the user running `ansible-playbook` must have **write permissions** to the specified `oscap_reports_store_path` on the Ansible control node.
* See `meta/main.yml` for the minimum Ansible version required.

Role Variables
--------------

### Primary Variables

These are the main variables you can set to control the scan, defined in `defaults/main.yml`.

| Variable | Description | Default |
| :--- | :--- | :--- |
| **`oscap_package`** | Manually specify the scanner package name. This bypasses OS-specific logic and is useful for unsupported systems. | `''` |
| **`oscap_content`** | Path to a **local** SCAP datastream file (on the controller) to be copied to the target. Required for unsupported systems | `''` |
| **`oscap_tailoring`** | Path to a **local** XCCDF tailoring file (on the controller) to be copied to the target. | `''` |
| **`oscap_profile`** | The specific profile ID to evaluate. Requird for unsupported systems | `''` |
| **`oscap_reports_store`** | If `true`, fetches the HTML and XML reports from the target back to the Ansible controller. | `false` |
| **`oscap_reports_store_path`** | Destination path on the **controller** for fetched reports. | 'files/scap-results/{{ ansible_date_time.iso8601_basic_short }}-{{ inventory_hostname }}/' |

---

### Internal OS-Specific Variables

These variables are loaded automatically from the `vars/` directory based on the target OS. You should not need to override these directly.

| Variable | Description |
| :--- | :--- |
| **`oscap_scanner_package`** | The default package name for the OpenSCAP scanner (e.g., `openscap-scanner`). |
| **`oscap_guide_package`** | The default package for the SCAP Security Guide (e.g., `scap-security-guide`). |
| **`oscap_guide_default_content`** | The absolute path to the default SCAP datastream on the target OS. |
| **`oscap_guide_default_profile`** | The default compliance profile used if `oscap_profile` is not set. |

Dependencies
------------

This role has no external dependencies on other Ansible Galaxy roles.

Example Playbook
----------------

### Example 1: Default Scan on Rocky Linux 9

This example runs the role on a group of Rocky Linux 9 servers. It uses all the built-in defaults for that OS and simply sets `oscap_reports_store` to `true` to fetch the results.

```yaml
- hosts: rockylinux_servers
  become: true
  roles:
    - { role: saintcore.oscap, oscap_reports_store: true }

```

### Example 2: Custom Scan on Unsupported OS (Ubuntu 24.04)

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
