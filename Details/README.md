# Zabbix Monitoring Setup

This document describes the complete Zabbix monitoring setup for the homelab, including agent configuration on monitored devices, Zabbix server installation and configuration, host addition, and custom dashboard creation.

**Image repository:** All screenshots referenced in this document are available at:  
`https://github.com/Danar2714/homelab-core-rack/tree/main/docs/Zabbix`

---

## A. Monitored Devices Setup

This section covers the installation and configuration of monitoring agents on each device in the homelab that will be monitored by Zabbix.

---

### 1. Backup Server – Zabbix Agent Configuration

#### 1.1 Installing Zabbix Agent 2 on Alpine

This Intel NUC Celeron runs Alpine Linux and acts as the backup server in the homelab. To integrate it into the Zabbix monitoring stack, **Zabbix Agent 2** is installed and configured to report to the DietPi Zabbix server at 172.16.0.5.

##### 1.1.1 Updating repositories and installing the agent

First, the Alpine package indexes are updated and the Zabbix Agent 2 packages are installed together with logrotate and the OpenRC service files.
```bash
apk update
apk add zabbix-agent2 zabbix-agent2-openrc logrotate
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/backupserver_install_zabbix_agent.png" width="50%" />

**Figure 1 – Installing Zabbix Agent 2 on Alpine**

The command `apk add` pulls zabbix-agent2, its OpenRC integration and the required libraries, preparing the system to run the agent as a managed service.

---

##### 1.1.2 Creating the log directory for Zabbix

By default, the agent expects a writable log directory under `/var/log/zabbix`. On this Alpine installation the directory is created manually and ownership is given to the zabbix user:
```bash
mkdir -p /var/log/zabbix
chown zabbix:zabbix /var/log/zabbix
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/backupserver_logs_zabbix_agent.png" width="50%" />

**Figure 2 – Preparing the Zabbix log directory**

This ensures that zabbix-agent2 can create and rotate its own log files without permission issues.

---

#### 1.2 Configuring Zabbix Agent 2

The main configuration file is `/etc/zabbix/zabbix_agent2.conf`. At minimum, the **Server**, **ServerActive** and **Hostname** parameters are adjusted so the agent can communicate with the central Zabbix server.

##### 1.2.1 Setting server and hostname parameters
```bash
nano /etc/zabbix/zabbix_agent2.conf
```

Relevant lines:
```text
Server=172.16.0.5
ServerActive=172.16.0.5
Hostname=Backup server
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/backupserver_config_zabbix_agent.png" width="80%" />

**Figure 3 – Zabbix agent configuration on the Backup Server**

- **Server** lists the IPs allowed to query the agent (passive checks).
- **ServerActive** defines where the agent should send active check data.
- **Hostname** must match the host name configured later in the Zabbix web interface (e.g. *Backup Server*).

---

#### 1.3 Enabling and starting the agent service

With the configuration in place, zabbix-agent2 is enabled in OpenRC so it starts at boot, and then started immediately.
```bash
rc-update add zabbix-agent2
rc-service zabbix-agent2 start
rc-service zabbix-agent2 status
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/backupserver_start_zabbix_agent.png" width="60%" />

**Figure 4 – Starting and enabling Zabbix Agent 2 on Alpine**

The `rc-update` command adds the service to the default runlevel, while `rc-service ... status` confirms that the agent is now running and ready to be discovered by the Zabbix server.

---

### 2. Proxmox Server – Zabbix Agent Configuration

#### 2.1 Installing Zabbix Agent 2 on Proxmox

This Intel NUC i5 runs Proxmox VE on top of Debian and acts as the main hypervisor in the homelab. To integrate it into the Zabbix monitoring stack, **Zabbix Agent 2** is installed and configured to report to the DietPi Zabbix server at 172.16.0.5.

##### 2.1.1 Enabling the Zabbix repository

First, the official Zabbix repository for Debian 12 is added so that the latest agent packages are available via APT.
```bash
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.0+debian12_all.deb
dpkg -i zabbix-release_latest_7.0+debian12_all.deb
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/proxmoxserver_install_zabbix_agent1.png" width="50%" />

**Figure 5 – Adding the Zabbix 7.0 repository on Proxmox**

The zabbix-release package drops the appropriate .list files under `/etc/apt/sources.list.d/`, allowing Proxmox to install Zabbix components directly from the upstream repository.

---

##### 2.1.2 Installing the agent package

With the repository configured, the Debian package index is refreshed (if needed) and **Zabbix Agent 2** is installed.
```bash
apt install zabbix-agent2 -y
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/proxmoxserver_install_zabbix_agent2.png" width="50%" />

**Figure 6 – Installing Zabbix Agent 2 on Proxmox**

The package installation also creates a zabbix-agent2 systemd service, which will later be enabled to start automatically with the hypervisor.

---

#### 2.2 Configuring Zabbix Agent 2

The main configuration file is `/etc/zabbix/zabbix_agent2.conf`. As with the backup server, the **Server**, **ServerActive** and **Hostname** parameters are adjusted so the agent can communicate with the central Zabbix server.

##### 2.2.1 Setting server and hostname parameters
```bash
nano /etc/zabbix/zabbix_agent2.conf
```

Relevant lines:
```text
Server=172.16.0.5
ServerActive=172.16.0.5
Hostname=Proxmox server
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/proxmoxserver_config_zabbix_agent.png" width="80%" />

**Figure 7 – Zabbix agent configuration on the Proxmox Server**

- **Server** lists the IPs allowed to query the agent (passive checks).
- **ServerActive** defines where the agent should send active check data.
- **Hostname** must match the host name configured later in the Zabbix web interface (e.g. *Proxmox Server*).

---

#### 2.3 Enabling and starting the agent service

With the configuration saved, the zabbix-agent2 service is enabled in systemd so it starts at boot, restarted to load the new configuration, and its status is verified.
```bash
systemctl enable zabbix-agent2
systemctl restart zabbix-agent2
systemctl status zabbix-agent2
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/proxmoxserver_start_zabbix_agent.png" width="80%" />

**Figure 8 – Starting and enabling Zabbix Agent 2 on Proxmox**

The status output confirms that the service is **active (running)** and using `/etc/zabbix/zabbix_agent2.conf`, meaning the hypervisor is now ready to be monitored by the Zabbix server.

---

### 3. Router hEX RB750Gr3 – SNMP Configuration

#### 3.1 Enabling SNMP Service

SNMP is enabled on the router to allow Zabbix to query system information, interface counters, and device health. The configuration is performed from **WinBox → IP → SNMP**.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/router_snmp_settings.png" width="50%" />

**Figure 9 – Enabling SNMP service and general settings**

The SNMP service is enabled, and optional fields such as *Contact* and *Location* are filled to provide identifying information in monitoring systems. Trap Version and other trap-related fields are left at default because Zabbix uses **polling**, not traps.

---

#### 3.2 Creating the SNMP Community

An SNMP community named **zabbix** is created. This acts as the "password" that allows the Zabbix server to read SNMP data from the router.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/router_snmp_community.png" width="55%" />

**Figure 10 – SNMP community configuration**

- **Name:** zabbix
- **Addresses:** 172.16.0.5 (IP of the Zabbix server)
- **Read Access:** enabled
- **Write Access:** disabled (recommended for security)
- **Security:** none (SNMPv2, no authentication)
- Authentication and encryption fields are unused unless SNMPv3 is configured.

Restricting access to the Zabbix server IP minimizes exposure of the SNMP service.

---

#### 3.3 Verifying SNMP Communities

The list of configured SNMP communities appears under **IP → SNMP → Communities**.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/router_snmp_communities.png" width="70%" />

**Figure 11 – SNMP communities overview**

The new community **zabbix** is listed with:
- **Address Filter:** 172.16.0.5
- **Security:** none
- **Read Access:** yes
- **Write Access:** no

This confirms that the router will accept SNMP requests **only** from the Zabbix server at 172.16.0.5, using the community string zabbix.

---

### 4. Switch RB260GS – SNMP Configuration

This section covers the SNMP setup required for Zabbix to monitor the **MikroTik RB260GS (SwOS)**. SNMP must be enabled and a community string configured so the Zabbix server can query metrics from the switch.

#### 4.1 Enabling SNMP and defining the community

SwOS exposes its SNMP configuration under the **SNMP** tab in the web interface.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/switch_snmp_settings.png" width="60%" />

**Figure 12 – SNMP configuration in SwOS**

The following parameters are configured:

- **Enabled:** SNMP service is turned on.
- **Community:** zabbix  
  This must match the community string configured in the Zabbix host settings.
- **Contact Info:** Admin  
  Informational field for device identification.
- **Location:** Homelab  
  Indicates the physical/logical placement of the device.

These values allow the Zabbix server at **172.16.0.5** (configured earlier in the host section) to access the switch metrics over SNMPv2. SwOS does not require specifying an SNMP allowed-address list; access control is handled implicitly via the LAN topology.

---

## B. Zabbix Server Setup

This section covers the installation and configuration of the Zabbix server on DietPi (Debian 13), including repository setup, database creation, initial web configuration, and the addition of monitored hosts.

---

### 1. Installing the Zabbix repository and packages

This section covers adding the official Zabbix repository on DietPi (Debian 13), updating the package lists and installing the Zabbix server, frontend and agent.

#### 1.1 Downloading the Zabbix repository package

The Zabbix repository .deb package is downloaded into `/root` using wget.
```bash
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_7.0-2+debian13_all.deb
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_install_zabbix_server1.png" alt="Downloading the Zabbix 7.0 repository package on DietPi" width="50%" />

**Figure 13 – Downloading the Zabbix repository package**

The command downloads `zabbix-release_7.0-2+debian13_all.deb` and the ls output confirms the file is present in the current directory.

---

#### 1.2 Installing the Zabbix repository

The downloaded package is installed with dpkg so that APT can use the Zabbix repository.
```bash
dpkg -i zabbix-release_7.0-2+debian13_all.deb
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_install_zabbix_server2.png" alt="Installing the Zabbix repository package with dpkg" width="50%" />

**Figure 14 – Enabling the Zabbix APT repository**

The system unpacks and sets up the zabbix-release package, which adds the Zabbix repository entries under `/etc/apt/sources.list.d/`.

---

#### 1.3 Updating APT indexes

After enabling the repository, the package lists are updated so the new Zabbix packages become available.
```bash
apt update
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_apt_update.png" alt="Running apt update after adding the Zabbix repository" width="50%" />

**Figure 15 – Updating package lists**

The output shows the standard Debian repositories plus the new repo.zabbix.com entries for Zabbix 7.0 on Debian Trixie.

---

#### 1.4 Installing Zabbix server, frontend and agent

With the repository configured, the Zabbix components and MariaDB server are installed in a single command.
```bash
apt install -y \
  mariadb-server \
  zabbix-server-mysql \
  zabbix-frontend-php \
  zabbix-apache-conf \
  zabbix-sql-scripts \
  zabbix-agent2
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_install_zabbix_server3.png" alt="Installing Zabbix server, frontend and agent with apt" width="50%" />

**Figure 16 – Installing Zabbix and MariaDB packages**

This installs the database server, Zabbix server daemon (MySQL/MariaDB backend), the PHP frontend with Apache integration, the SQL schema scripts and the Zabbix Agent 2.

---

### 2. Securing the MariaDB server

Before creating the Zabbix database, the MariaDB instance is hardened using the mariadb-secure-installation helper.
```bash
mariadb-secure-installation
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_mariadb1.png" alt="Running mariadb-secure-installation and setting the root password" width="50%" />

**Figure 17 – Initial MariaDB secure installation steps**

During the wizard, a root password is set and Unix socket authentication options are confirmed according to the prompts.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_mariadb2.png" alt="Removing anonymous users, test database and reloading privileges in MariaDB" width="50%" />

**Figure 18 – Removing test database and tightening access**

Anonymous users are removed, remote root login is disabled, the default test database is dropped and the privilege tables are reloaded to apply the changes.

---

### 3. Creating the Zabbix database and user

Once MariaDB is secured, the database and user for Zabbix are created. First, the MariaDB shell is opened as root:
```bash
mariadb -u root -p
```

Inside the MariaDB prompt, the following SQL statements are executed:
```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_mariadb_db_creation.png" alt="Creating the Zabbix database and user in MariaDB" width="50%" />

**Figure 19 – Creating the Zabbix database and granting privileges**

The database **zabbix** is created with UTF-8 settings, a local user **zabbix** is defined with an (initially) empty password, and full privileges on the zabbix schema are granted. In a real deployment, a strong password should be configured instead of an empty value.

---

### 4. Importing the initial Zabbix schema

Zabbix provides a pre-built schema and data file in compressed form. It is imported into the zabbix database using zcat piped into mariadb:
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz \
  | mariadb -u zabbix -p zabbix
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_schema.png" alt="Importing the Zabbix schema into MariaDB using zcat and mariadb" width="50%" />

**Figure 20 – Importing the Zabbix SQL schema**

The command expands `server.sql.gz` and feeds it directly to MariaDB, populating all required tables and initial data in the zabbix database.

---

### 5. Configuring the Zabbix server to use MariaDB

The Zabbix server configuration file is edited so the daemon knows how to connect to the MariaDB instance.
```bash
nano /etc/zabbix/zabbix_server.conf
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_server_config1.png" alt="Opening the Zabbix server configuration file in nano" width="50%" />

**Figure 21 – Zabbix server configuration file**

The file `/etc/zabbix/zabbix_server.conf` contains the main server parameters, including database connection options. In the database section, the **DBUser** and **DBPassword** parameters are set to match the MariaDB user created earlier. For documentation purposes, the password field is left empty:
```text
DBUser=zabbix
DBPassword=
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_server_config2.png" alt="Setting DBUser and empty DBPassword in zabbix_server.conf" width="50%" />

**Figure 22 – Database credentials in zabbix_server.conf**

**DBUser** is already set to zabbix by default; only **DBPassword** needs to be uncommented. A real deployment should use a strong password here and in the corresponding MariaDB user configuration.

---

### 6. Starting and enabling Zabbix services

Once the database configuration is complete and `zabbix_server.conf` points to the correct credentials, the web server and Zabbix daemons are restarted and enabled to start on boot.

#### 6.1 Restarting Apache

After installing the PHP frontend, Apache is restarted so it loads the Zabbix virtual host and PHP configuration.
```bash
systemctl restart apache2
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_apache_restart.png" width="50%" />

**Figure 23 – Restarting Apache after Zabbix frontend installation**

---

#### 6.2 Restarting and checking the Zabbix server
```bash
systemctl restart zabbix-server
systemctl status zabbix-server
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_server_status.png" width="50%" />

**Figure 24 – Verifying that the Zabbix server is running**

---

#### 6.3 Restarting and checking the Zabbix Agent 2
```bash
systemctl restart zabbix-agent2
systemctl status zabbix-agent2
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_agent_status.png" width="50%" />

**Figure 25 – Verifying that Zabbix Agent 2 is running**

---

#### 6.4 Enabling services on boot
```bash
systemctl enable zabbix-server zabbix-agent2 apache2 mariadb
```

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_services_enable.png" width="50%" />

**Figure 26 – Enabling Zabbix-related services at boot**

---

### 7. Initial Zabbix web configuration

With all services running, the first login to the Zabbix web interface completes the installation through a guided wizard.

#### 7.1 Welcome screen and language selection

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_initial_ui.png" width="50%" />

**Figure 27 – Zabbix 7.0 installer welcome page**

---

#### 7.2 Checking PHP pre-requisites

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_prerequisites_ui.png" width="50%" />

**Figure 28 – PHP prerequisites check**

---

#### 7.3 Configuring the database connection

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_db_connection_ui.png" width="50%" />

**Figure 29 – Database connection parameters**

---

#### 7.4 Setting server details and time zone

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_settings_ui.png" width="50%" />

**Figure 30 – Zabbix server general settings**

---

#### 7.5 Pre-installation summary

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_preinstall_ui.png" width="50%" />

**Figure 31 – Pre-installation configuration summary**

---

### 8. Post‑installation configuration: Housekeeping and dashboard verification

After completing the initial setup, Zabbix provides several administrative options to control database growth, data retention, and overall system behavior. One of the recommended steps is to verify **Housekeeping** settings and confirm that the system dashboard loads correctly.

#### 8.1 Accessing Housekeeping settings

Housekeeping controls how long Zabbix stores historical data, events, and internal logs. To access it:

**Administration → Housekeeping**

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_housekeeping_ui1.png" width="40%" />

**Figure 32 – Navigating to Housekeeping settings**

This menu contains retention periods for events, alerts, services, user sessions and discovery data.

---

#### 8.2 Reviewing default data retention periods

The Housekeeping page displays multiple retention values, which can be customized depending on storage capacity and required historical depth.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_housekeeping_ui2.png" width="50%" />

**Figure 33 – Default Housekeeping retention values**

By default, Zabbix stores events (90 days), service data (7 days), internal data (7 days), discovery results (1 day) and user sessions (30 days). Lowering these values can reduce database size on small systems like Raspberry Pi.

---

### 9. Verifying system status on the global dashboard

Once the installation is complete, Zabbix loads the **Global view** dashboard, summarizing metrics such as host availability, event counts, CPU usage, and server health.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_zabbix_result.png" width="90%" />

**Figure 34 – Zabbix Global Dashboard after installation**

The dashboard confirms:
- Zabbix server is running
- Frontend is operational
- Hosts are being monitored
- Basic telemetry (CPU, availability, problems by severity) is functional

This indicates the installation and configuration were successful.

---

### 10. Adding monitored hosts to Zabbix

Once the Zabbix server and dashboard are fully operational, the next step is to add the homelab servers as monitored hosts. **This assumes the Zabbix Agent 2 is already installed and running on each server**.

Additionally, **this assumes that SNMP is already configured on the network devices**.

---

#### 10.1 Adding the Backup Server (Intel NUC Celeron)

To register the Backup Server in Zabbix:

1. Go to **Configuration → Hosts → Create host**.
2. Enter the hostname and visible name (for example, Backup Server).
3. Add the template **Linux by Zabbix agent**.
4. Assign the host to the **Linux servers** group.
5. Add an Agent interface pointing to the LAN IP of the backup NUC (e.g. 172.16.0.3) on port **10050**.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_add_backuphost1.png" width="80%" />

**Figure 35 – Creating the Backup Server host entry**

The backup server is added with the standard Linux agent template, which provides OS-level metrics such as CPU, memory, filesystem usage, and basic networking.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_add_backuphost2.png" width="90%" />

**Figure 36 – Backup Server monitored successfully**

Once the agent responds, the host status changes to **Available**, and Zabbix starts collecting metrics and populating items, triggers, and graphs for this server.

---

#### 10.2 Adding the Proxmox Server (Intel NUC i5)

The same process is used to add the Proxmox hypervisor:

1. Go to **Configuration → Hosts → Create host**.
2. Set the hostname and visible name (for example, Proxmox Server).
3. Add the template **Linux by Zabbix agent**.
4. Assign the host to the **Hypervisors** group.
5. Configure the Agent interface with the Proxmox LAN IP (e.g. 172.16.0.4) on port **10050**.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_add_proxmoxhost1.png" width="80%" />

**Figure 37 – Creating the Proxmox Server host entry**

Grouping the Proxmox node under *Hypervisors* keeps virtualization infrastructure separated from generic Linux servers in the Zabbix UI.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_add_proxmoxhost2.png" width="90%" />

**Figure 38 – Proxmox Server monitored successfully**

After the agent connection is established, the Proxmox host also appears as **Available**, confirming that the Zabbix server can reach and monitor the hypervisor over the homelab network.

---

#### 10.3 Adding the Router hEX RB750Gr3 (Mikrotik)

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_add_router1.png" width="80%" />

**Figure 39 – Creating the Router hEX RB750Gr3 entry**

The router is added using an **SNMP interface**, since Mikrotik devices expose metrics through their SNMP service.

- Template used: **Mikrotik by SNMP**
- Host group: **Network devices**
- SNMP version: **SNMPv2**
- SNMP community: zabbix (must match the router configuration)

This configuration allows Zabbix to retrieve system information, interface statistics, CPU usage and traffic counters.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_add_router2.png" width="90%" />

**Figure 40 – Router monitored successfully**

Once communication is established, the host shows **SNMP availability** in green. Zabbix begins populating items and graphing interface traffic, CPU load and device health.

---

#### 10.4 Adding the Switch RB260GS

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_add_switch1.png" width="80%" />

**Figure 41 – Creating the Switch RB260GS entry**

The RB260GS switch is also monitored via SNMP.

- Template: **Mikrotik by SNMP**
- Host group: **Network devices**
- Interface: 172.16.0.2 on port **161**
- SNMP community: zabbix

Since RB260GS is a simple Smart Switch, SNMP provides metrics such as interface counters, port status and link speeds.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/rpi_add_switch2.png" width="90%" />

**Figure 42 – Switch monitored successfully**

The switch appears as **Available (SNMP)**, confirming Zabbix can poll it successfully. Graphing and item updates begin immediately using the Mikrotik SNMP template.

---

## C. Dashboards

Once the Zabbix server is fully configured and all hosts are added (backup server, Proxmox hypervisor, router and switch), the next step is to build dashboards that provide a quick overview of the homelab status. Dashboards are created from **Monitoring → Dashboards** and can contain widgets such as host availability, problems list, graphs, and resource usage.

For this homelab, three dashboards were created:

- **Main** – Global overview of the rack and recent problems.
- **Network devices** – Focused on the router and switch.
- **Servers** – Focused on the backup and Proxmox servers, plus the Zabbix server itself.

Each dashboard is composed only of built‑in widgets, so it can be recreated easily on any Zabbix installation.

---

### 1. Main dashboard – global rack overview

The **Main** dashboard aggregates the most important information about the entire rack:

- Host availability by host group (Hypervisors, Linux servers, Network devices, Zabbix servers).
- Problems by severity.
- A navigator panel with shortcuts to each monitored host.
- A problems table showing recent issues such as restarts or time sync warnings.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/main_dashboard.png" width="90%" />

**Figure 43 – Main dashboard with global availability and recent problems**

From this screen it is possible to see at a glance whether any host in the rack has issues, how many are available, and which problems have occurred recently.

---

### 2. Network devices dashboard

The **Network devices** dashboard focuses on the router hEX RB750Gr3 and the RB260GS switch. Its goal is to monitor link status, interface traffic and basic health (disk and memory) for the network layer of the homelab.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/network_devices_dashboard1.png" width="90%" />

**Figure 44 – Network devices overview: availability, router disk and memory**

The top widgets show the availability of network devices and an empty problems panel (when there are no active issues). Two pie charts display total vs. used disk and memory on the router.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/network_devices_dashboard2.png" width="90%" />

**Figure 45 – Router interfaces and traffic graphs**

This part of the dashboard shows a horizontal summary of router interfaces (Bridge-Lan, E1–E5) and detailed graphs for the LAN bridge and the primary uplink. Each graph displays bits received/sent and potential errors or discarded packets.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/network_devices_dashboard3.png" width="90%" />

**Figure 46 – Switch interfaces and per‑port traffic**

The switch section shows the status of ports P1–P5 and SFP, along with dedicated graphs for selected ports. This makes it easy to verify which physical ports are in use and to detect abnormal traffic patterns or link failures.

---

### 3. Servers dashboard

The **Servers** dashboard concentrates on the backup SERVER, the Proxmox hypervisor and the Zabbix server. It provides CPU, memory, disk usage and network traffic for each node, allowing a quick check of resource utilization.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/servers_dashboard1.png" width="90%" />

**Figure 47 – Servers overview and Backup Server resources**

The top widgets summarize host availability for the servers group and show overall CPU, memory and disk usage of the backup SERVER. Below, interface graphs display LAN and WAN traffic for this host.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/servers_dashboard2.png" width="90%" />

**Figure 48 – Proxmox Server resources and VM interface traffic**

This section shows Proxmox resource usage (CPU, memory and disks) and traffic graphs for the physical interface (eno1) and the main virtual bridge (vmbr0). These graphs help verify both hypervisor load and traffic generated by virtual machines.

<img src="https://raw.githubusercontent.com/Danar2714/homelab-core-rack/main/docs/Zabbix/servers_dashboard3.png" width="90%" />

**Figure 49 – Zabbix Server resources and Tailscale traffic**

The last part of the dashboard monitors the Zabbix server itself (DietPi on Raspberry Pi 3 B+), including CPU, memory and SD card usage. Separate graphs show traffic on the LAN interface (eth0) and the Tailscale interface (tailscale0), confirming that monitoring data and remote administration traffic are flowing correctly.

---


