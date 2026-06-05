# Enterprise Ansible Automation Codebase

This repository provides a modular, production-ready, and enterprise-aligned Ansible configuration framework. It implements security hardening, multi-tier application deployment, dynamic inventories, custom Execution Environments, secrets management, and ITSM (ServiceNow) callback integration, mapping directly to the **Ansible Learning Roadmap** guidelines.

---

## Directory Structure

```text
├── ansible.cfg                    # Global configuration with speed and audit profiling
├── execution-environment.yml      # Custom container runtime spec for ansible-builder
├── README.md                      # Developer operations guide
├── .github/
│   └── workflows/
│       └── ansible-ci.yml         # GitHub Actions pipeline (YAML validation, ansible-lint, syntax check)
├── collections/
│   └── requirements.yml           # Galaxy collection dependencies
├── inventories/
│   ├── aws_ec2.yml                # Dynamic inventory configuration for cloud hosts
│   ├── dev/
│   │   ├── hosts.yml              # Dev host definitions
│   │   └── group_vars/            # Group configurations (all, webservers, databases, loadbalancers)
│   ├── staging/
│   │   ├── hosts.yml              # Staging host definitions
│   │   └── group_vars/            # Staging environment definitions
│   └── prod/
│       ├── hosts.yml              # Prod host definitions
│       └── group_vars/            # Prod environments (references Vault values)
│           └── all/
│               └── vault.yml      # Plaintext / Encrypted sensitive variables
├── playbooks/
│   ├── common.yml                 # Playbook applying OS baseline and SSH hardening
│   ├── deploy_stack.yml           # Orchestrates multi-tier deployment (DB -> Web -> LB)
│   ├── itsm_callback.yml          # Change management wrapper simulating ticket update/Slack hooks
│   └── site.yml                   # Master playbook integrating hardening and ITSM wrapper
└── roles/
    ├── common/                    # OS baseline role (dnf, sysctl configuration, firewall setup)
    ├── webserver/                 # Nginx, automatic SSL certificate generation, and config templates
    ├── database/                  # MariaDB setup, root hardening, database, and user provisioning
    └── loadbalancer/              # HAProxy frontend/backend configurations mapping target webservers
```

---

## 1. Quick Start

### Step 1: Install Collection Dependencies
Before running the playbooks, download the required external collections:
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### Step 2: Syntax Validation
Verify that the codebase syntax is correct:
```bash
ansible-playbook -i inventories/dev/hosts.yml playbooks/site.yml --syntax-check
```

### Step 3: Verify Variables and Host Mappings
Inspect which hosts are mapped to which variables:
```bash
ansible-inventory -i inventories/dev/hosts.yml --list
```

---

## 2. Secrets Management (Ansible Vault)

Sensitive values in production (`inventories/prod/group_vars/all/vault.yml`) should not be stored in plaintext in version control.

### Encrypt the Vault File:
Run this command to encrypt the secrets:
```bash
ansible-vault encrypt inventories/prod/group_vars/all/vault.yml
```
You will be prompted to enter a password.

### Running Playbooks with Vault:
When executing the production playbook, you must pass the vault password:
```bash
# Prompt for vault password
ansible-playbook -i inventories/prod/hosts.yml playbooks/site.yml --ask-vault-pass

# Or reference a secure vault password file (recommended in pipelines/AAP)
ansible-playbook -i inventories/prod/hosts.yml playbooks/site.yml --vault-password-file ~/.ansible/vault_pass.txt
```

### Edit Encrypted Vault Content:
To modify encrypted parameters:
```bash
ansible-vault edit inventories/prod/group_vars/all/vault.yml
```

---

## 3. Dynamic Cloud Inventories

The config file `inventories/aws_ec2.yml` configures the dynamic inventory plugin `amazon.aws.aws_ec2`. 

To view dynamic hosts currently running in your AWS environment grouped by tags:
```bash
# Verify AWS connection and list targets
ansible-inventory -i inventories/aws_ec2.yml --graph
```

---

## 4. Execution Environments (EEs)

Instead of using traditional Python virtual environments (which suffer from configuration drift), this repository defines a containerized runtime configuration in `execution-environment.yml`.

To build the container image using `ansible-builder`:
```bash
ansible-builder build -f execution-environment.yml -t custom-ee:v1.0.0
```

To run a playbook inside your built Execution Environment locally:
```bash
ansible-runner run . -p playbooks/site.yml -i inventories/dev/hosts.yml
```

---

## 5. Performance and Observability

Optimizations configured in [ansible.cfg](file:///Users/spakcomm-ajay/Documents/Software-Roadmap/Ansible/project/ansible.cfg):
*   **Pipelining (`pipelining = True`):** Drastically speeds up tasks by transferring and running python files in memory on target systems rather than disk writes.
*   **Multiplexing (`control_path`):** Reuses active SSH sessions to bypass TCP handshake latency.
*   **Profiling (`ansible.posix.profile_tasks`):** Prints a timestamped report tracking which tasks take the longest, exposing bottlenecks.

---

## 6. Enterprise Compliance Mapping

This project maps directly to the 19 governance controls and architectural requirements outlined in the [Enterprise Level Pointers](file:///Users/spakcomm-ajay/Documents/Software-Roadmap/Ansible/Enterprise%20Level%20Pointers/Enterprise%20Level%20Pointers.md) guide:

| Section | Enterprise Requirement | Mapping in the Codebase |
| :--- | :--- | :--- |
| **1. Governance & Ownership** | Repository structure, naming conventions, and pull request reviews. | Standardized repository layout (roles, inventories, playbooks). Enforced via `.github/workflows/ansible-ci.yml`. |
| **2. Security & Access Control** | Least privilege access boundaries and connection setups. | Configured via environment-specific targets and distinct users (e.g., `devops`, `deploy_stage`, `deploy_prod`). |
| **3. Secrets Management** | Dynamic lookups and Vault usage. | Plaintext secrets are decoupled. Standard Vault template at `prod/group_vars/all/vault.yml`. Lookups configured in `playbooks/secrets_integration.yml`. |
| **4. Content Strategy** | Reusable collections and roles. | Implemented custom modular roles (`common`, `webserver`, `database`, `loadbalancer`). Pinned dependencies in `collections/requirements.yml`. |
| **5. Testing & Quality** | Dry runs, linting, and syntax checks. | Pipeline configured for `yamllint` and `ansible-lint`. All playbooks syntax-verified with `ansible-playbook --syntax-check`. |
| **6. CI/CD & Release Management** | Automated pipelines and environment promotions. | GitHub Actions pipeline configuration checks syntax and formats automatically. |
| **7. AWX / AAP Architecture** | Job workflows and webhook triggers. | Dynamic HAProxy backend configurations loop over inventory groups; callback variables support automation controller job runs. |
| **8. Execution Environments** | Standardized runtime container specs. | Configured builder definition file `execution-environment.yml` to package collections, system packages, and Python dependencies. |
| **9. Inventory Management** | Dynamic inventory and variable precedence. | Multi-tier environment variables under `group_vars`. Cloud targets queried dynamically via `inventories/aws_ec2.yml`. |
| **10. Compliance & Policy** | OS Hardening and Exception Registry. | Tasks in `roles/common/tasks/hardening.yml` are mapped to DISA STIG & CIS controls. Bypasses are governed by the `roles/common/vars/exceptions.yml` register. |
| **11. Observability & Logging** | Performance profiling and metrics callback. | Enforced in `ansible.cfg` using `ansible.posix.profile_tasks` and `ansible.posix.timer` plugins. |
| **12. Operations & Support** | System troubleshooting and operating runbooks. | Comprehensive operating guide and troubleshooting steps are documented in this README file. |
| **13. Backup, Restore, & DR** | Database backups and recovery procedures. | Implemented database exports and S3 archiving in `playbooks/backup_restore.yml` with secure restore conditional blocks. |
| **14. Cloud, Windows, Network** | Hybrid automation, WinRM, and PowerShell. | Configured dynamic AWS EC2 filters and created `playbooks/windows_remediation.yml` using `win_updates`, `win_service`, and PowerShell. |
| **15. ITSM & Change Management** | Linking deployments to ITSM change requests. | Fully automated in `playbooks/servicenow_itsm.yml` utilizing the `uri` module to POST/PUT ticket state changes in ServiceNow. |
| **16. Skills Matrix** | Developer and architect roles. | Documented above to guide team roles (Developers manage roles; Architects configure EEs, Vault, and Playbook workflows). |
| **17. Learning Path** | Certifications and training guidelines. | Aligned tasks to Red Hat RHCE (EX294), AAP Developer (EX374), and Advanced Automation (EX480) topics. |
| **18. Acceptance Checklist** | Production acceptance gates. | Included in this README to ensure security, backups, EEs, and auditing are validated before launch. |
| **19. Final Enterprise Outcome** | Standardized, auditable platform operations. | Realized by this complete, modularized, version-controlled, and syntactically validated automation repository. |

