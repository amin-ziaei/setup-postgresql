# Ansible — PostgreSQL Setup with Monitoring

Role-based Ansible playbook for installing and configuring PostgreSQL and setting up PostgreSQL monitoring with `postgres_exporter` for Prometheus.

## Project Structure

```
setup-postgress/
├── site.yml                        # Main playbook entry point
├── requirements.yml                # Required Ansible collections
└── roles/
    ├── postgresql/
    │   ├── defaults/main.yml       # Default PostgreSQL variables
    │   ├── tasks/
    │   │   ├── main.yml
    │   │   ├── install.yml         # Install packages from the PGDG repository
    │   │   ├── configure.yml       # Manage postgresql.conf and pg_hba.conf
    │   │   └── users_and_dbs.yml   # Create PostgreSQL users and databases
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── postgresql.conf.j2
    │       └── pg_hba.conf.j2
    └── monitoring/
        ├── defaults/main.yml       # Default monitoring variables
        ├── tasks/
        │   ├── main.yml
        │   ├── postgres_exporter.yml # Install postgres_exporter
        │   └── firewall.yml        # Open the exporter port with UFW, if enabled
        ├── handlers/main.yml
        └── templates/
            ├── postgres_exporter.service.j2
            └── postgres_exporter.env.j2
```

## Prerequisites

```bash
# Install required collections
ansible-galaxy collection install -r requirements.yml
```

## Configuration Before Running

### 1. Define the inventory outside this project

This project does not include an internal inventory file. Create your own inventory separately and pass it with `-i` when running the playbook:

```ini
[postgresql_servers]
db01 ansible_host=192.168.1.10 ansible_user=ubuntu ansible_become=true
```

### 2. Configure variables

All default values are defined in `roles/postgresql/defaults/main.yml` and
`roles/monitoring/defaults/main.yml`. Override them in your external inventory,
with a custom vars file, or with `--extra-vars`:
- Database name and database user
- Database user password and monitoring user password
- Memory values based on server RAM
- Allowed addresses in `pg_hba.conf`
- PostgreSQL version

Main application database variables:

```yaml
postgresql_app_database_name: myapp_db
postgresql_app_database_user: myapp_user
postgresql_app_database_password: "ChangeMe_AppPassword!"
```

Main monitoring variables:

```yaml
postgres_monitor_user: postgres_exporter_monitor
postgres_monitor_password: "ChangeMe_MonitorPassword!"
```

For real environments, override these values in your external inventory or your own vars file.

## Run

```bash
# Full run (PostgreSQL + Monitoring)
ansible-playbook -i /path/to/inventory.ini site.yml

# PostgreSQL only
ansible-playbook -i /path/to/inventory.ini site.yml --tags postgresql

# Monitoring only
ansible-playbook -i /path/to/inventory.ini site.yml --tags monitoring

# PostgreSQL configuration only, without reinstalling packages
ansible-playbook -i /path/to/inventory.ini site.yml --tags configure

# Dry run without applying changes
ansible-playbook -i /path/to/inventory.ini site.yml --check --diff
```

## Monitoring Endpoints

After a successful run:

| Service | Port | Address |
|-------|------|------|
| PostgreSQL | 5432 | `db01:5432` |
| PostgreSQL Exporter | 9187 | `http://db01:9187/metrics` |

### Prometheus scrape config

```yaml
scrape_configs:
  - job_name: 'postgresql'
    static_configs:
      - targets: ['db01:9187']
```

## Memory Tuning by RAM

| Server RAM | shared_buffers | effective_cache_size | work_mem |
|----------|---------------|---------------------|---------|
| 2 GB     | 512MB         | 1536MB              | 4MB     |
| 4 GB     | 1GB           | 3GB                 | 8MB     |
| 8 GB     | 2GB           | 6GB                 | 16MB    |
| 16 GB    | 4GB           | 12GB                | 32MB    |
| 32 GB    | 8GB           | 24GB                | 64MB    |
