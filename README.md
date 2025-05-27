# Ansible role to deploy TPV Broker

A role that installs the [TPV Broker](https://github.com/usegalaxy-eu/tpv-broker), a FastAPI application for advanced metascheduling.

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
    # defaults file for tpv_broker
    tpv_broker_dir: "/opt/tpv_broker"
    tpv_broker_repo: "https://github.com/usegalaxy-eu/tpv-broker.git"
    tpv_broker_commit_id: "main"
    tpv_broker_force_checkout: true # discards local changes if any
    tpv_broker_venv_dir: "/opt/tpv_broker/venv"
    tpv_broker_virtualenv_command: "python -m venv"
    tpv_broker_systemd_service_name: "tpv_broker"
    tpv_broker_systemd_service_user: "nginx"
    tpv_broker_ssl_certificate: ""
    tpv_broker_ssl_certificate_key: ""
    tpv_broker_nginx_conf: true
    tpv_broker_proxy_config:
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
      - role: usegalaxy_eu.tpv_broker
        src: https://github.com/usegalaxy-eu/ansible-tpv-broker.git
```

Additonally, the role requires setting up the necessary nginx location as follows:

```jinja
server {
...

  {% if tpv_broker_nginx_conf %}
	location {{ tpv_broker_proxy_config.location }} {
		rewrite  ^{{ tpv_broker_proxy_config.location }}(.*)  /$1 break;
        proxy_pass {{ tpv_broker_proxy_config.proxy_pass_protocol }}://{{ tpv_broker_proxy_config.proxy_pass_host }}:{{ tpv_broker_proxy_config.proxy_pass_port }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
	}

	location {{ tpv_broker_proxy_config.location }}version {
		alias {{ tpv_broker_dir }}/version;
		add_header Content-Type text/plain;
        }
	{% endif %}

}

...
```

And a TPV rank function to send the requests to the TPV Broker endpoint:

```yaml
users:
  esg_users:
    abstract: true
    params:
    scheduling:
      prefer:
        - be-pulsar
        - sanjay-pulsar
        - de-pulsar
        - cz-pulsar
        - it-pulsar
        - sp-pulsar
        - uk-pulsar
        - tr-pulsar
        - nor-pulsar
        - pl-pulsar
        - ch-pulsar
        - sl-pulsar
    rank: |
      import sys
      import requests
      import json
      import pathlib
      from ruamel.yaml import YAML
      import logging
    
      # Set up logging
      logging.basicConfig(stream=sys.stdout, level=logging.DEBUG)                       
      log = logging.getLogger(__name__)
        
      # Load static objectstore info 
      # NOTE: currently object store info is stored in a yaml
      objectstore_loc_path = "{{ tpv_config_dir }}/object_store_locations.yml"
      dataset_attributes = helpers.get_dataset_attributes(job.input_datasets)

      yaml=YAML(typ='safe')
      objectstore_file = pathlib.Path(objectstore_loc_path)
      objectstore_dict = yaml.load(objectstore_file)

      # Format the destination data
      candidate_destinations_list = [{"id": dest.id, **dest.context} for dest in candidate_destinations if dest.context and 'latitude' in dest.context and 'longitude' in dest.context]
      log.debug(f"candidate_destinations_list: {candidate_destinations_list}")
      
      # Static job info
      if not entity.gpus:
        gpus = 0
      job_info = {"tool_id": tool.id, "mem": mem, "cores": cores, "gpus": gpus,}

      # URL of the FastAPI application
      api_url = "{{ inventory_hostname }}/{{ tpv_broker_proxy_config.location }}/process_data"

      input_data = {
        "destinations": candidate_destinations_list,
        "objectstores": objectstore_dict,
        "datasets": dataset_attributes,
        "job_info": job_info,
      }

      # Send a POST request to the API endpoint
      response = requests.post(api_url, json=input_data)

      # Check if the request was successful (status code 200)
      if response.status_code == 200:
          result = response.json()
      else:
          log.debug(f"Request failed with status code {response.status_code}: {response.text}")
          print(f"Request failed with status code {response.status_code}: {response.text}")

      log.debug(f"sorted destinations: {result}")
      # log.debug(f"cand destinations: {[dest.id for dest in candidate_destinations]}")
      # Sort the candidate destinations based on the sorted_destinations from the result
      final_destinations = sorted(
          (dest for dest in candidate_destinations if str(dest.id) in result["sorted_destinations"]),
          key=lambda obj: result["sorted_destinations"].index(str(obj.id))
      )

      final_destinations

  user01:
    inherits: esg_users
```

## License

See [LICENSE.md](LICENSE.md)

