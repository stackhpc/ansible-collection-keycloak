This role deploys Keycloak in standalone-HA configuration, using MariaDB as the backend storage 

### Role variables

#### OCI image
##### Optional
* **keycloak_docker_name:** Name of the Keycloak docker container (default: "keycloak")
* **keycloak_docker_image:** Docker image to use (default: "quay.io/keycloak/keycloak")
* **keycloak_docker_tag:** Image tag to use (default: "21.1")
* **keycloak_container.etc_hosts:** Dict of host-to-IP mappings (default: {})
 
#### Keycloak configuration
These configuration options broadly follow those documented by the [Keycloak project](https://www.keycloak.org/server/all-config).
##### Mandatory
* **keycloak_admin_password:** Password for the Keycloak admin user
* **keycloak_db_password:** Database password
* **keycloak_db_url_host:** Database host
* **keycloak_db_login_username:** Database user used to create Keycloak database (if `keycloak_db_ensure_database: true`)
* **keycloak_db_login_password:** Password for database user used to create Keycloak database (if `keycloak_db_ensure_database: true`)

##### Optional
* **keycloak_admin_username:** Admin username for Keycloak (default: `"admin"`)
* **keycloak_db_type:** Database type (only mariadb tested, default: `"mariadb"`)
* **keycloak_db_ensure_database:** Ensure that the Keycloak database and user exists (default: `true`, requires
  `keycloak_db_login_username` and `keycloak_db_login_password`)
* **keycloak_db_mariadb_user_priv:** MySQL privileges string to set when ensuring Keycloak database user (default: `"'{{ keycloak_db_url_database }}.*:ALL'"`)
* **keycloak_db_mariadb_user_host:** The "host" part of the MySQL username when ensuring Keycloak database user
  (default: `"%"`, all hosts)
* **keycloak_db_username:** Database username (default: `"keycloak"`)
* **keycloak_db_url_database:** Database name (default: `"keycloak"`)
* **keycloak_db_url_port:** Database port (default: `3306`)
* **keycloak_transaction_xa_enabled:** Enable XA transaction support (default: `false`)
* **keycloak_http_bind_interface:** Interface to bind the Keycloak http server (default: `"lo"`)
* **keycloak_http_port:** Port to bind the keycloak http server (default: `"8080"`)
* **keycloak_metrics_enabled:** Enable Prometheus metrics (default: `false`)
* **keycloak_health_enabled:** Expose Keycloak health check endpoints (default: `false`)
* **keycloak_log_level:** Keycloak log level (default: unset - OCI image default)
* **keycloak_hostname_url:** Keycloak base URL for frontend URLs, including scheme, host, port and path (default: unset - OCI image default)
* **keycloak_hostname_strict:** Disables dynamically resolving the hostname from request headers (default: unset - OCI image default)
* **keycloak_hostname_path:** This should be set if proxy uses a different context-path for Keycloak (default: unset - OCI image default)
* **keycloak_http_relative_path:** Set the path relative to `/` for serving resources (default: unset)


#### Keycloak JGroups cache configuration
##### Optional
* **keycloak_jgroups_discovery_protocol:** JGroups cache discovery protocol (only JDBC_PING tested, default: `"JDBC_PING"`)
* **keycloak_jdbc_driver_name:** JGroups cache JDBC driver name (only org.mariadb.jdbc.Driver tested, default: `"org.mariadb.jdbc.Driver"`)
* **keycloak_jdbc_initialise_sql:** JGroups cache initialise SQL statement, default: 
```SQL
    CREATE TABLE IF NOT EXISTS JGROUPSPING 
    (own_addr varchar(200) NOT NULL, 
    cluster_name varchar(200) NOT NULL, 
    ping_data VARBINARY(255), 
    constraint PK_JGROUPSPING 
    PRIMARY KEY (own_addr, cluster_name));
```
* **keycloak_jgroups_discovery_bind_interface:** JGroups discovery bind interface (should be an internal network,
  default: `"lo"`)
* **keycloak_jgroups_discovery_bind_port:** JGroups discovery bind port (default: `7800`)
* **keycloak_jgroups_fd_port:** JGroups failure-detection port (default: `57800`)

#### Keycloak container command
##### Optional
* **keycloak_server_command:** Keycloak server command (default: `"start"`)
* **keycloak_proxy_type:** Keycloak proxy type (Default: `""`)
* **keycloak_server_command_args_extra:** Extra args to pass to the Keycloak server command (default: `""`)

#### Cache configuration files
##### Optional
* **keycloak_host_config_dir:** Host configuration directory (default: `"/etc/keycloak"`)
* **keycloak_cache_config_file:** Cache configuration filename (default: `"cache-ispn-config.xml"`)
* **keycloak_cache_config_file_template:** Cache configuration template file (default: `"cache-ispn-jdbc-ping.xml.j2"`)

#### Docker volumes
##### Optional
* **keycloak_volumes_extra:** A list of extra volumes to be specified (default: `[]`)


Example playbook (used with OpenStack Kayobe)

```YAML
---
- name: Run Keycloak Install role
  hosts: controllers
  vars:
    keycloak:
      keycloak_admin_username: keycloak_admin
      keycloak_admin_password: keycloak_password
      keycloak_database_password: keycloak_database_password
  tasks:
    - name: Set a fact about the virtualenv on the remote system
      set_fact:
        virtualenv: "{{ ansible_python_interpreter | dirname | dirname }}"
      when:
        - ansible_python_interpreter is defined
        - not ansible_python_interpreter.startswith('/bin/')
        - not ansible_python_interpreter.startswith('/usr/bin/')
    
    - name: Ensure Python PyMySQL module is installed
      pip:
        name: pymysql
        extra_args: "{% if pip_upper_constraints_file %}-c {{ pip_upper_constraints_file }}{% endif %}"
        virtualenv: "{{ virtualenv is defined | ternary(virtualenv, omit) }}"
      become: "{{ virtualenv is not defined }}"
    
    - import_role: 
        name: stackhpc.keycloak.deploy
      vars:
        keycloak_admin_username: "{{ keycloak.keycloak_admin_username }}"
        keycloak_admin_password: "{{ keycloak.keycloak_admin_password }}"
        keycloak_db_url_host: "{{ database_address | put_address_in_context('url') }}"
        keycloak_db_url_database: keycloak
        keycloak_db_password: "{{ keycloak.keycloak_database_password }}"
        keycloak_db_login_user: root
        keycloak_db_login_password: "{{ database_password }}" 
        keycloak_db_user: keycloak
        keycloak_jgroups_discovery_bind_interface: ens3
        keycloak_proxy_type: edge # Behind HAProxy
        keycloak_http_bind_interface: ens3
        keycloak_hostname_strict: "true"
        keycloak_hostname_url: "https://identity.example.com"
```