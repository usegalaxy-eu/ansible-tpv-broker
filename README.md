# Ansible role: tpv-metascheduler-api

A role that installs tpv-metascheduler-api.

## Requirements

No specific requirements, the role is self-contained.

## Role variables

Role variables are documented in the forms of comments on [defaults/main.yml](defaults/main.yml)

## Dependencies

None.

## Example Playbook

```yaml
---
- name: Install and configure tpv api
  hosts: all
  vars:
    # defaults file for tpv_metascheduler_api
    tpv_metascheduler_api_dir: "/opt/tpv_metascheduler_api"
    tpv_metascheduler_api_repo: "https://github.com/usegalaxy-eu/tpv-metascheduler-api.git"
    tpv_metascheduler_api_commit_id: "main"
    tpv_metascheduler_api_force_checkout: true # discards local changes if any
    tpv_metascheduler_api_venv_dir: "/opt/tpv_metascheduler_api/venv"
    tpv_metascheduler_api_virtualenv_command: "python -m venv"
    tpv_metascheduler_api_systemd_service_name: "tpv_metascheduler_api"
    tpv_metascheduler_api_systemd_service_user: "nginx"
    tpv_metascheduler_api_ssl_certificate: ""
    tpv_metascheduler_api_ssl_certificate_key: ""
    tpv_metascheduler_api_nginx_conf: true
    tpv_metascheduler_api_proxy_config:
    location: "/tpv-api/"
    proxy_pass_host: "127.0.0.1"
    proxy_pass_port: 8000
    proxy_pass_protocol: "http"

    influx_address: "{{ inventory_hostname }}"
    influx_port: 8086
    influx_db: galaxy
    influx_username: "influx_user"
    influx_password: "influx_pass"
  roles:
      # Fastapi that collects stats about Pulsar nodes
      - role: pdg.tpv_metascheduler_api
```

## License

See [LICENSE.md](LICENSE.md)

