# ImmuneML Galaxy Playbook

Deploy and configure an **ImmuneML-enabled Galaxy instance** using Ansible.

This repository provides an orchestration layer on top of the baseline Galaxy deployment role. It installs Galaxy when needed, configures Galaxy-native Conda dependency resolution, registers immuneML Galaxy tools, and applies ImmuneML-specific UI/static assets.

---

## Repository Contents

```text
.
├── immuneml.yml
├── group_vars/
│   └── galaxyservers.yml
├── roles/
│   └── galaxy_deployment/
├── files/
│   ├── welcome_immuneml.html
│   └── welcome_immuneml_files/
└── templates/
    ├── dependency_resolvers_conf.xml.j2
    ├── tool_conf_empty.xml.j2
    └── immuneml_toolbox_block.xml.j2
```

Main files:

- `immuneml.yml`  
  Main orchestrator playbook.

- `group_vars/galaxyservers.yml`  
  Deployment configuration and helper variables.

- `roles/galaxy_deployment`  
  Baseline Galaxy deployment role. Always runs on every playbook execution (Play 3 is unconditional).

- `files/`  
  Local welcome page and static Galaxy assets.

- `templates/`  
  Local Ansible templates used to render Galaxy configuration fragments.

---

## What `immuneml.yml` Does

The playbook runs in five plays.

---

### 1. Baseline Status Check (diagnostic only)

Collects Galaxy state information for reporting purposes.

It checks:

- Galaxy server directory
- Galaxy config file
- Galaxy systemd service unit
- Galaxy service state
- Galaxy API endpoint
- PostgreSQL database connection config

It then sets `galaxy_needs_baseline_attention: true` **unconditionally**, so the baseline role always runs regardless of the current Galaxy health state.

---

### 2. Bootstrap Minimal Galaxy Config

Ensures the Galaxy config directory exists and creates a minimal `tool_conf.xml` if missing.

---

### 3. Galaxy Baseline Deployment (always runs)

Runs the `galaxy_deployment` role on **every execution**, unconditionally.

Galaxy baseline is always redeployed regardless of whether Galaxy is already running or healthy.

---

### 4. ImmuneML Runtime Strategy Configuration

Configures Galaxy to use immuneML through Galaxy-native Conda/Bioconda dependency resolution.

This stage:

- Enforces:

```yaml
immuneml_runtime_strategy: conda
```

- Creates or updates:

```text
dependency_resolvers_conf.xml
```

- Uses the Conda channels:

```text
bioconda
conda-forge
defaults
```

- Wires the dependency resolver config into Galaxy config.

The dependency resolver file is rendered from:

```text
templates/dependency_resolvers_conf.xml.j2
```

---

### 5. ImmuneML Tool and UI Registration

Registers immuneML tools and UI assets in Galaxy.

This stage:

- Clones and installs official immuneML Galaxy wrappers.
- Uses wrapper-defined Conda requirements from the immuneML tools repository.

- Registers immuneML tools in `tool_conf.xml`.
- Registers `yaml`, `yml`, and immuneML receptor datatypes in `datatypes_conf.xml`.
- Copies local ImmuneML welcome/static assets from `files/`.
- Sets the Galaxy `welcome_url`.
- Restarts Galaxy and nginx.
- Performs post-restart Galaxy API health checks.

---

## Main Configuration

Most configuration lives in:

```text
group_vars/galaxyservers.yml
```

Update this file before deployment.

---

### Target VM

```yaml
galaxy:
  host_ip: "galaxy_immune_nrec"
  ssh_user: "ubuntu"
```

- `galaxy.host_ip`  
  Inventory host, IP, or SSH alias.

- `galaxy.ssh_user`  
  SSH user used by Ansible.

---

### Galaxy Source

```yaml
galaxy_repo: "https://github.com/galaxyproject/galaxy.git"
galaxy_commit_id: "release_26.0"
galaxy_force_checkout: true
```

---

### Galaxy Admin and Database

```yaml
galaxy_admin_email: "admin@example.com"
galaxy_database_password: "CHANGE_ME_STRONG_PASSWORD"
```

Before production use:

- replace `galaxy_admin_email`
- replace `galaxy_database_password`

---

### HTTPS

Example test configuration:

```yaml
galaxy_enable_https: true
galaxy_https_mode: "selfsigned"
galaxy_server_name: "galaxy.local"
galaxy_https_redirect: false
galaxy_https_validate_certs: false
```

Notes:

- `galaxy_https_validate_certs: false` is needed for self-signed certificate health checks.
- For production, use a real DNS name and proper TLS certificates.

---

### ImmuneML Runtime

```yaml
immuneml_runtime_strategy: "conda"
immuneml_cli_runtime_enabled: false
```

Galaxy tools should use Conda/Bioconda dependency resolution.

The playbook intentionally avoids installing immuneML with `pip` inside Galaxy tool environments.

---

### ImmuneML Galaxy Wrappers

```yaml
immuneml_tools_repo_url: "https://github.com/uio-bmi/immuneml_tools.git"
immuneml_tools_version: "update-immuneml-3-0-27"
immuneml_tools_source_dir: "/srv/galaxy/immuneml_tools_source"
immuneml_tools_force_refresh: true
```

---

### ImmuneML Wrapper Compatibility

```yaml
# Leave unset unless you intentionally patch prod_macros.xml.
# immuneml_setuptools_version: "<81"
```

By default, wrapper dependency compatibility is taken from the wrapper repo (`prod_macros.xml`).

---

### ImmuneML Welcome and Static Assets

```yaml
immuneml_welcome_folder: "files"
immuneml_welcome_html: "welcome_immuneml.html"
```

The playbook copies local files from:

```text
files/
```

into Galaxy static assets.

Example expected files:

```text
files/
├── welcome_immuneml.html
└── welcome_immuneml_files/
```

Optional favicon overrides can be configured via:

```yaml
# galaxy_favicon_ico_src: "files/favicon.ico"
# galaxy_favicon_svg_src: "files/favicon.svg"
```

---

### Galaxy Branding

```yaml
galaxy_brand: "UiO Galaxy"
galaxy_page_title: "ImmuneML Galaxy"
```

---

## Helper Variables

`group_vars/galaxyservers.yml` also defines resolved helper variables used by the playbook.

Examples:

```yaml
galaxy_root_resolved
galaxy_server_dir_resolved
galaxy_config_dir_resolved
galaxy_config_file_resolved
galaxy_service_unit_resolved
galaxy_runtime_user_resolved
galaxy_runtime_group_resolved
galaxy_dependency_resolvers_conf
galaxy_conda_prefix_resolved
```

These keep the playbook readable and avoid relying on role defaults before the role is loaded.

---

## Templates

The playbook uses local Ansible templates from:

```text
templates/
```

Current templates:

```text
templates/dependency_resolvers_conf.xml.j2
templates/tool_conf_empty.xml.j2
templates/immuneml_toolbox_block.xml.j2
```

### `dependency_resolvers_conf.xml.j2`

Renders Galaxy Conda dependency resolver configuration.

### `tool_conf_empty.xml.j2`

Creates a minimal Galaxy toolbox config if one does not exist.

### `immuneml_toolbox_block.xml.j2`

Defines the immuneML tool menu sections inserted into Galaxy `tool_conf.xml`.

---

## Quick Start

### 1. Prepare inventory

If you use `./deploy.sh`, this step is optional because the script generates `hosts` from `group_vars/galaxyservers.yml` automatically.

Create or update:

```text
hosts
```

Example:

```ini
[galaxyservers]
galaxy ansible_host=galaxy_immune_nrec ansible_user=ubuntu
```

---

### 2. Update deployment configuration

Edit:

```text
group_vars/galaxyservers.yml
```

At minimum, update:

```yaml
galaxy:
  host_ip: "your-host-or-ip"
  ssh_user: "ubuntu"

galaxy_admin_email: "your-admin@example.org"
galaxy_database_password: "replace-with-a-strong-password"
```

---

### 3. Install Ansible roles

If using `deploy.sh`, choose the role install option.

Or run manually:

```bash
ansible-galaxy install -r requirements.yml -p roles
```

---

### 4. Run the playbook

From the repository root:

```bash
ansible-playbook -i hosts immuneml.yml
```

Or use the deployment helper:

```bash
./deploy.sh
```

Then choose:

```text
6) Run ImmuneML playbook only
```

---

## Useful Checks

### Check inventory variables

```bash
ansible galaxyservers -i hosts -m debug -a "var=galaxy_server_dir_resolved"
```

### Check playbook syntax

```bash
ansible-playbook -i hosts immuneml.yml --syntax-check
```

### Check Galaxy API on the VM

HTTP:

```bash
curl -sf http://127.0.0.1/api/version
```

HTTPS with self-signed certificate:

```bash
curl -k -sf https://127.0.0.1/api/version
```

### Check Galaxy service

```bash
sudo systemctl status galaxy --no-pager
```

### Check PostgreSQL configuration

```bash
sudo grep -nE "database_connection|postgresql" /srv/galaxy/config/galaxy.yml
```

### Check immuneML wrapper requirements

```bash
sudo grep -nE "requirement|immuneML|setuptools" /srv/galaxy/server/tools/immuneml/prod_macros.xml
```

---

## Wipe and Redeploy Notes

If `/srv/galaxy` is a mounted disk or mountpoint, wipe operations should delete the contents inside `/srv/galaxy` but keep the `/srv/galaxy` directory/mountpoint itself.

Preferred cleanup pattern:

```bash
sudo find /srv/galaxy -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +
```

Avoid removing the mountpoint directly:

```bash
sudo rm -rf /srv/galaxy
```

---

## Disk Space Reminder

Galaxy and Conda environments can use significant disk space.

Make sure the target VM has enough storage and that the Galaxy root path is available:

```text
/srv/galaxy
```

For production or larger tests, provision enough disk space and mount it at:

```text
/srv/galaxy
```

---

## Notes

- The playbook avoids direct edits to Galaxy core source.
- Galaxy baseline deployment always runs unconditionally on every playbook execution (Play 3).
- immuneML Galaxy tools use Galaxy-native Conda/Bioconda dependency resolution.
- The welcome page and static assets are copied from the local `files/` directory.
- The tool menu is managed through `tool_conf.xml`.
- Datatypes are managed through `datatypes_conf.xml`.

---

## License

This project is licensed under the Apache License 2.0. See `LICENSE`.
