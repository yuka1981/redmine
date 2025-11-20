# Redmine Installation Playbook

This repository automates a production-ready Redmine 6.1.0 node on Rocky Linux (or RHEL-compatible) hosts. It compiles Ruby 3.4.7 under `/opt/ruby/3.4.7`, installs PostgreSQL 16 locally, serves Redmine through Puma + nginx, wires Redis/Sidekiq for background jobs, and keeps SELinux enforcing with proper labels.

## Repository Structure

- `ansible.yml` – single entry-point that imports `playbooks/site.yml`.
- `playbooks/` – inventory, `group_vars/`, and the roles that express each layer (common packages, ruby, postgres, redmine, puma, nginx, selinux, redis, sidekiq).

## Requirements

1. **Control node**: Ansible 2.16+ with `community.postgresql`, `community.general`, and `ansible.posix` collections.
   ```bash
   ansible-galaxy collection install community.postgresql community.general ansible.posix
   ```
2. **Target nodes**: Rocky Linux 10 (or compatible) with SSH access as a sudo-capable user. The playbook expects passwordless sudo and will reboot services but not the host.
3. **Networking**: Allow outbound HTTPS for package downloads and inbound TCP/80 for nginx (open 443 manually if you add TLS).

## Quick Start

1. **Clone & inventory**

   ```bash
    git clone <this_repo> redmine-installation
    cd redmine-installation
    cp playbooks/inventory.ini.example playbooks/inventory.ini
   ```

   Replace the placeholders in `playbooks/inventory.ini` with each Redmine host's SSH details.

2. **Configure variables** – Adjust `playbooks/group_vars/all.yml` if you need different versions, paths, or database credentials. Defaults build Ruby to `/opt/ruby/3.4.7`, Redmine to `/opt/redmine/6.1.0`, and configure PostgreSQL with `redmine`/`QCT@Admin`.

3. **Run the playbook**
   ```bash
   cd playbooks
   ansible-playbook -i inventory.ini ../ansible.yml
   ```
   Use `--tags '<role>'` to re-run a narrow portion (e.g., `--tags redmine,sidekiq`).

## What the Playbook Does

- **common** (`roles/common`): Enables the Rocky CRB repo, installs EPEL, compiler toolchains, image libraries, SCM tools, and ensures `/opt/ruby` + `/opt/redmine` exist.

- **ruby**: Downloads Ruby 3.4.7 source, builds it with `--disable-install-doc`, and installs Bundler in `/opt/ruby/3.4.7`.

- **postgres**: Installs PostgreSQL 16 from the OS repos, initializes the cluster, enforces MD5 auth in `pg_hba.conf`, configures the `postgres` password (`postgres_superuser_password`), and creates the `redmine` role, schema, and database.

- **redmine**: Creates the `redmine` system user, downloads `redmine-6.1.0.tar.gz`, lays out runtime directories, manages key configs (`database.yml`, `configuration.yml`, `active_job_queue_adapter.rb`, `sidekiq.yml`), installs gems to `vendor/bundle`, seeds secrets, runs migrations, and loads default data (honoring `redmine_default_lang`).

- **redis**: Builds Redis 8.4.0 from source into `/opt/redis/<version>`, provisions config/data directories, and enables a custom `redis.service` bound to `127.0.0.1:6379`.

- **puma**: Templates `config/puma.rb`, deploys `redmine-puma.service`, and ensures Puma runs under the Redmine account using the compiled Ruby toolchain.

- **sidekiq**: Drops `redmine-sidekiq.service` so background jobs use Redis, then enables and starts it.

- **nginx**: Installs nginx, publishes `redmine.conf` pointing to the Puma Unix socket in `tmp/sockets/puma.sock`, validates the config (`nginx -t`), and restarts the service.

- **selinux**: Keeps SELinux enforcing, labels Puma sockets/pids/log directories, ships an SELinux module so nginx (httpd_t) can launch Puma through a wrapper, and restarts services to apply contexts.

## Customization

Edit `playbooks/group_vars/all.yml` for:

- `ruby_version`, `ruby_prefix_base`
- `redmine_version`, `redmine_root_base`, `redmine_default_lang`
- DB credentials (`redmine_db_*`, `postgres_superuser_password`)
- Redis parameters (`redis_version`, ports, directories)

To override values per-host, create host/group var files that Ansible will load automatically.

## Verification & Operations

- Verify services: `sudo systemctl status redmine-puma redmine-sidekiq redis nginx postgresql`.
- Confirm SELinux contexts: `sudo ls -Z /opt/redmine/6.1.0/tmp/sockets`.
- Check Redmine: visit `http://<server>` and log in with the default admin account (`admin`/`admin`, then force a password change).
- Logs live under `/opt/redmine/6.1.0/log` and `/var/log/nginx`.

Re-running `ansible-playbook` is safe; tasks are idempotent. Use tags when applying fixes (e.g., `--tags postgres` after editing DB vars). For TLS, extend the nginx role with certificates and reload nginx.
