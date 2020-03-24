Ansible-Elasticsearch-Cluster
=========
#### It is highly recommended that you do not directly access this repository from your CI/CD jobs. Make a fork in your project's SCM instead. By doing so, you make sure that your cluster will not be broken in case one of the new commits appear to have a bug. You can keep the fork synchronized with the origin manually

#### References
* Elasticsearch Roles: https://www.elastic.co/guide/en/elasticsearch/reference/7.3/defining-roles.html
* Kibana Roles (just in case): https://www.elastic.co/guide/en/kibana/7.3/role-management-api-put.html
* Elasticsearch Security: https://www.elastic.co/guide/en/elasticsearch/reference/7.3/security-settings.html
* Elasticsearch LDAP Config (paid ES subscription required): https://www.elastic.co/guide/en/elasticsearch/reference/7.3/ldap-realm.html
* Elasticsearch LDAP Role Mapping: https://www.elastic.co/guide/en/elasticsearch/reference/7.3/security-api-put-role-mapping.html

## SETUP GUIDE

### Preparation steps

* Generate a Root CA certificate

```
$ openssl req -x509 -newkey rsa:4096 -keyout root-ca-key.pem -out root-ca-cert.pem -days 3650 -nodes -subj "/C=US/ST=NY/L=NewYork/O=MyCompany/OU=IT/CN=MyDomain/emailAddress=admin@example.com"
```

* Convert the certificate and key to `base64` and add them to, say, `elasticsearch-secrets-production.yml`, and put that file in `{{ playbook_dir }}/vars`

```
$ cat root-ca-cert.pem | base64 -w 0
$ cat root-ca-key.pem | base64 -w 0
```

```
elasticsearch_root_ca_cert_base64: <ROOT_CA_CERT_BASE64_ENCODED>
elasticsearch_root_ca_key_base64: <ROOT_CA_KEY_BASE64_ENCODED>
```

* Generate a password for the default Elasticsearch user and add it to `{{ playbook_dir }}/vars/elasticsearch-secrets-production.yml`

```
elasticsearch_bootstrap_password: <STRONG_PASSWORD_FOR_ELASTICSEARCH>
```

* Encrypt the secrets file

```
$ ansible-vault encrypt vars/elasticsearch-secrets-production.yml
```

* Create an inventory (an Ansible inventory file should only contain the ES nodes belonging to the same cluster)

`FILE: {{ playbook_dir }}/inventory-elasticsearch_production`

```
[elasticsearch_production]
elasticsearch-coord-prod-1 ansible_host=10.236.0.1 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=coord-only
elasticsearch-coord-prod-2 ansible_host=10.236.0.2 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=coord-only
elasticsearch-data-prod-1 ansible_host=10.236.0.10 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=master-data
elasticsearch-data-prod-2 ansible_host=10.236.0.11 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=master-data
elasticsearch-data-prod-3 ansible_host=10.236.0.12 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=master-data
elasticsearch-data-prod-4 ansible_host=10.236.0.13 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=master-data
elasticsearch-data-prod-5 ansible_host=10.236.0.14 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=master-data

[kibana_production]
elasticsearch-coord-prod-1 ansible_host=10.236.0.1 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=coord-only
elasticsearch-coord-prod-2 ansible_host=10.236.0.2 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=coord-only
```

Available options for `es_node_role`:

```
master-only        # dedicated Master-eligible node
data-only          # dedicated Data processing node
ingest-only        # dedicated Ingest node
ml-only            # dedicated Machine learning node
coord-only         # Coordinating only node
```

### Ansible requirements

* `FILE: {{ playbook_dir }}/requirements/elasticsearch-requirements.yml`

```
---
# Basic server setup
- name: ansible-basic-server-setup
  src: git@github.com:arsenii-stefanov/ansible-basic-server-setup.git
  scm: git
  version: master
# Set up Python
- name: ansible-python
  src: git@github.com:arsenii-stefanov/ansible-python.git
  scm: git
  version: master
# Set up Docker
- name: ansible-docker
  src: git@github.com:arsenii-stefanov/ansible-docker.git
  scm: git
  version: master
# Set up Elasticsearch
- name: ansible-elasticsearch-cluster
  src: git@github.com:arsenii-stefanov/ansible-elasticsearch-cluster.git
  scm: git
  version: master
- name: ansible-kibana
  src: git@github.com:arsenii-stefanov/ansible-kibana.git
  scm: git
  version: master
```

### Ansible variables example

* `FILE: {{ playbook_dir }}/vars/elasticsearch-secrets-production.yml`

```
elasticsearch_bootstrap_password: <STRONG_PASSWORD_FOR_ELASTICSEARCH>

elasticsearch_root_ca_cert_base64: <ROOT_CA_CERT_BASE64_ENCODED>
elasticsearch_root_ca_key_base64: <ROOT_CA_KEY_BASE64_ENCODED>

# These are built-in users; adding a built-in user here will change its password; you can find their names in the Elasticsearch docs
elasticsearch_built_in_users:
  kibana:
    password: "<your strong password>"

# Adding a user 'elasticsearch_users' will create the user if it does not exist or update the existing user's config 
elasticsearch_users:
  john-doe:
    password: "<your strong password>"
    full_name: "John Doe"
    roles: [ "elasticsearch_administrators", "monitoring_user" ]
  jane-doe:
    password: "<your strong password>"
    full_name: "Jane Doe"
    roles: [ "elasticsearch_developers" ]

elasticsearch_users_delete:
- old-user
- test-user
```

* `FILE: {{ playbook_dir }}/vars/elasticsearch-production.yml`

```
---
########################################
##     START ELASTICSEARCH CONFIG     ##
########################################
elasticsearch_installation_type: "docker"        # Available options: docker

elasticsearch_main_mount_dir: "/srv"

elasticsearch_disk_mounts: [
    {
      src: "/dev/sdb",
      dest: "{{ elasticsearch_main_mount_dir }}",
      fstype: "ext4",
      fs_opts: "-m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard",
      mount_opts: "discard,defaults,nofail",
      state: mounted
    }
]

elasticsearch_cluster_name: "elasticsearch-production-cluster"

elasticsearch_version: "7.3.2"

# If you would like to install Filebeat for collecting ES logs and sending them to either this very same Elasticsearch or to another Elasticsearch, or Graylog
install_filebeat_logger: true

# Logstash (Graylog) example
filebeat_config_opts:
 output_type: "logstash"
 output_hosts: ["graylog.example.local:12201"]

# Elasticsearch example (Make sure you define the corresponding role 'elasticsearch_logger' in the 'elasticsearch_roles' variable. See the config below)
filebeat_config_opts:
 output_type: "elasticsearch"
 output_hosts: ["elasticsearch.example.local:9200"]
 output_user: <elasticsearch log user>
 output_password: <elasticsearch log password>



elasticsearch_openssl_csr:
  common_name: "elasticsearch.local"
  country_name: "US"
  state_or_province_name: "NY"
  locality_name: "NY"
  organizational_unit_name: "IT"
  organization_name: "MyOrganization"
  email_address: "admin@elasticsearch.local"
  subject_alt_name:
    - "DNS:{{ inventory_hostname }}"
    - "IP:{{ ansible_host }}"

# You may add env vars here
elasticsearch_env_vars:
  # JVM heap size should NOT exceed 50% of your physical RAM
  ES_JAVA_OPTS: "-Xms14g -Xmx14g"
  MAX_LOCKED_MEMORY: "unlimited"
  ELASTIC_PASSWORD: "{{ elasticsearch_bootstrap_password }}" # 'elasticsearch_bootstrap_password' should be defined in an Ansible secrets file

# These settings will be used in Jinja2 templates
elasticsearch_config_opts:
  number_of_shards: "5"
  number_of_replicas: "1"
  backup_repo: ""
  minimum_master_nodes: "2" # This setting is required if you use an Elasticsearch version older than 7.x
  json_logs: true

# The entire dictionary will be added to the 'xpack' dictionary in the 'templates/elasticsearch/elasticsearch.yml.j2' as is
elasticsearch_xpack_config:
  license:
    self_generated:
      type: "basic"
  monitoring:
    collection:
      enabled: true
  security:
    enabled: true
    transport:
      ssl:
        enabled: true
        verification_mode: "certificate"
        certificate_authorities: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_root_ca_cert_name }}"
        certificate: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_node_cert_name }}"
        key: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_node_key_name }}"
    http:
      ssl:
        enabled: false
        certificate_authorities: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_root_ca_cert_name }}"
        certificate: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_node_cert_name }}"
        key: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_node_key_name }}"
    authc:
      realms:
        native:
          realm1:
            order: 0
            authentication:
              enabled: true
        ldap:
          ldap1:
            order: 1
            url: "ldap://ldap.example.local:636"
            bind_dn: "userid=elasticsearch, ou=serviceAccounts, dc=ldap, dc=example, dc=local"
            user_dn_templates:
              - "cn={0}, ou=users, dc=ldap, dc=example, dc=local"
            group_search:
              base_dn: "ou=groups, dc=ldap, dc=example, dc=local"
            unmapped_groups_as_roles: false

# Elasticsearch Post-Installation Tasks custom templates example

### You can adjust basic configs by specifying variables, such as 'elasticsearch_roles', for example:

elasticsearch_roles:
  elasticsearch_administrators:
    cluster: ["all"]
    indices:
      - allow_restricted_indices: true
        names: ["*"]
        privileges: ["all"]
    applications:
      - application: "kibana-.kibana"
        privileges: ["all"]
        resources: ["*"]
    run_as: []
    transient_metadata:
      enabled: "true"
  elasticsearch_developers:
    cluster:
      - monitor
      - manage_pipeline
      - manage_ingest_pipelines
      - transport_client
      - monitor_ml
      - monitor_data_frame_transforms
      - monitor_watcher
      - read_ccr
      - read_ilm
      - monitor_rollup
      - create_snapshot
    indices:
      - allow_restricted_indices: false
        names: ["*"]
        privileges: ["all"]
    applications:
      - application: "kibana-.kibana"
        privileges: ["all"]
        resources: ["*"]
    run_as: []
    transient_metadata:
      enabled: "true"
  # https://www.elastic.co/guide/en/beats/libbeat/current/troubleshooting-upgrade.html
  elasticsearch_logger:
    cluster:
      - monitor
      - manage_ilm
      - manage_index_templates
      - manage_ingest_pipelines
    # https://www.elastic.co/guide/en/beats/filebeat/7.3/command-line-options.html#setup-command
    applications:
      - application: "kibana-.kibana"
        privileges: ["all"]
        resources: ["*"]
    indices:
      - allow_restricted_indices: false
        names: ["filebeat-*"]
        privileges: ["all"]
  elasticsearch_reader_bot:
    indices:
      - allow_restricted_indices: false
        names: ["*pattern1*","*pattern2*"]
        privileges: ["read"]
  elasticsearch_writer_bot:
    indices:
      - allow_restricted_indices: false
        names: ["*pattern1*","*pattern2*"]
        privileges: ["all"]

### Also, you can use custom JSON files for ES configs other then ES users and roles

elasticsearch_config_files_dir: "{{ playbook_dir }}/config-files/elasticsearch-production"

elasticsearch_config_files: [
  { method: "POST", es_endpoint: "/_security/role/elasticsearch_administrators", es_file: "{{ elasticsearch_config_files_dir }}/roles/elasticsearch_administrators.json" },
  { method: "POST", es_endpoint: "/_security/role/elasticsearch_developers", es_file: "{{ elasticsearch_config_files_dir }}/roles/elasticsearch_developers.json" },
  { method: "POST", es_endpoint: "/_security/role/elasticsearch_managers", es_file: "{{ elasticsearch_config_files_dir }}/roles/elasticsearch_managers.json" }
]

######################################
##     END ELASTICSEARCH CONFIG     ##
######################################

#############################################
###                DOCKER                 ###
#############################################
docker_packages_ubuntu: [ "docker-ce" ]

#############################################
###                PYTHON                 ###
#############################################

ansible_python_interpreter: /usr/bin/python3
python_packages: [ python3-pip, python3-numpy, python3-scipy ]

python_modules_docker: [
  { name: "docker", state: "latest", extra_args: "--user", executable: "pip3" }
]
```

* Enabling XPack and Generating SSL certificates for Elasticsearch Transport (port 9300 used for node interconnection)

If you would like to enable XPack and have SSL certs generated, make sure you have the following variable `elasticsearch_config['xpack']['security']['enabled']` set to `true` in your Ansible vars file. Note that this is not just a variable but part of the dictionary `elasticsearch_config` which is your main config (see the example above)

* `FILE: {{ playbook_dir }}/vault.py`

```
#!/usr/bin/env python

import os

print os.environ['VAULT_PASS']
```

* ES JSON config files

```
{{ playbook_dir }}/
├── config-files
    ├── elasticsearch-production
        |
        └── roles
            ├── elasticsearch_administrators.json
            ├── elasticsearch_developers.json
            └── elasticsearch_managers.json
```

* `FILE: {{ playbook_dir }}/playbook.yml`

```
# ELASTICSEARCH
- hosts: elasticsearch_production
  connection: ssh
  become: true
  become_user: root
  become_method: sudo
  any_errors_fatal: true
  gather_facts: true
  serial: 1

  pre_tasks:

    - name: Include Variables
      include_vars: "{{ item }}"
      with_items:
        - "elasticsearch-production.yml"
        - "elasticsearch-secrets-production.yml"
      tags: [always]

  roles:
    - { role: "ansible-basic-server-setup", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - { role: "ansible-python", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - { role: "ansible-docker", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - ansible-elasticsearch-cluster

# KIBANA
- hosts: kibana_production
  connection: ssh
  become: true
  become_user: root
  become_method: sudo
  any_errors_fatal: true
  gather_facts: true
  serial: 1

  pre_tasks:

    - name: Include Variables
      include_vars: "{{ item }}"
      with_items:
        - "kibana-secrets-production.yml"
        - "kibana-production.yml"
      tags: [always]

  roles:
    - ansible-kibana
```

#### Run Ansible

* Install Ansible Galaxy requirements

```
ansible-galaxy -vvvv install -f -r requirements/elasticsearch-requirements.yml
```
 * Export  the Ansible Vault password
```
export VAULT_PASS=<Vault password>
```

* Perform initial setup of an Elasticsearch cluster (with `elasticsearch_initial_setup: true` your ES will not be restarted upon updating the config files)

```
ansible-playbook --vault-password-file ./vault.py -i inventory-elasticsearch_production elasticsearch-production.yml -e '{"elasticsearch_initial_setup": true}'
```
* Run post installation tasks (these tasks mostly interact with the ES API, you can create users, roles, templates, etc. so these actions do not require a restart)

```
ansible-playbook --vault-password-file ./vault.py -i inventory-elasticsearch_production elasticsearch-production.yml --tags "run_post_installation_tasks"
```

* Post installation tasks [Filebeat Logger] (requires Kibana to be up and running, so this task should be run after Kibana, roles and users are configured)

```
ansible-playbook --vault-password-file ./vault.py -i inventory-elasticsearch_production elasticsearch-production.yml --tags "run_filebeat_logger_tasks"
```

* Second run when adding a new ES node

```
ansible-playbook --vault-password-file ./vault.py -i inventory-elasticsearch_production elasticsearch-production.yml
```

* Second run on the same cluster (if you need to update some config files, directories), this will skip the basic server setup steps

```
ansible-playbook --vault-password-file ./vault.py -i inventory-elasticsearch_production elasticsearch-production.yml -e '{"skip_basic_server_setup": true}'
```
