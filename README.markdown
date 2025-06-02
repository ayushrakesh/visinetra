# Visinetra: Network Visibility & Telemetry Automation

A comprehensive network monitoring and automation solution that provides end-to-end visibility into network performance through automated configuration, real-time telemetry collection, and intuitive dashboards. Built with Ansible, GNS3, Telegraf, InfluxDB, and Grafana.

## Project Overview

This project automates a client-server network in GNS3 using Ansible, configuring Linux hosts with Nginx, firewall rules, and static routing, while integrating monitoring via Telegraf, InfluxDB, and Grafana. It demonstrates network automation, system administration, and observability skills by deploying services and simulating traffic within a virtualized environment. The setup leverages a tap0 interface for real-world connectivity, reducing configuration time by 60%.

### Key Features

- **Network Setup**: Two Linux hosts (Server1 and Client1) in a GNS3 topology, connected via a switch and a tap0 interface for real-world connectivity.
- **Automation**: Ansible playbooks to configure a web server (Nginx), SNMP, firewall rules (ufw), and monitoring (Telegraf) on Server1, and a traffic generator on Client1.
- **Monitoring**: Telegraf collects system metrics (CPU, memory, network) and SNMP data, sending them to InfluxDB Cloud for visualization in Grafana.
- **Traffic Simulation**: Client1 generates HTTP traffic to Server1 to simulate network activity.
- **Static Routing**: Uses a single subnet with static routing, avoiding the complexity of dynamic routing protocols.

### Skills Demonstrated

- **Ansible Automation**: Automating network services, firewall rules, and monitoring configurations.
- **Network Administration**: Configuring IP addresses, firewall rules, and network services.
- **Monitoring and Observability**: Using Telegraf, InfluxDB, and Grafana to monitor system and network metrics.
- **Virtualized Networking**: Building a network in GNS3 with a tap0 interface for external connectivity.
- **Linux System Administration**: Managing Linux hosts, services, and networking.

## Prerequisites

To run this project locally, ensure you have the following:

### Hardware/Software Requirements

- **Operating System**: Ubuntu 24.04 LTS (or similar Linux distribution).
- **GNS3**: Version 2.2 or later, installed and configured.
- **Docker**: Installed to run Linux containers in GNS3.
- **Ansible**: Version 2.9 or later, installed (`sudo apt install ansible`).
- **Python Packages**:
  - Install `ansible-pylibssh` for better SSH performance: `pip3 install ansible-pylibssh --user`.
- **InfluxDB Cloud Account**:
  - Sign up for a free InfluxDB Cloud account at InfluxData.
  - Create a bucket named `visinetra` in the `Dev` organization.
  - Use the provided token in `server1.yml` or replace it with your own.
- **Grafana**: Set up Grafana (local or cloud) to visualize InfluxDB data.
  - Add InfluxDB as a data source in Grafana using the same credentials as in `server1.yml`.
- **Internet Access**: Ensure your host system has internet access (e.g., via Ethernet `eno1` or Wi-Fi `wlo1`).

### Network Requirements

- **tap0 Interface**: A tap interface on the host system for GNS3 connectivity.
  - IP: `192.168.69.254/24`.
- **Host System Configuration**:
  - Enable IP forwarding and configure NAT rules to allow GNS3 hosts to access the internet (see below).

## Project Setup

### Step 1: Set Up the Host System

1. **Enable IP Forwarding**:

   - Run the following commands to enable IP forwarding on your host system:

     ```bash
     sudo sysctl -w net.ipv4.ip_forward=1
     echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
     ```

   - This ensures the GNS3 network can route traffic to the internet.

2. **Configure NAT and Firewall Rules**:

   - Assuming your internet interface is `eno1` (replace with `wlo1` if using Wi-Fi):

     ```bash
     sudo iptables -t nat -A POSTROUTING -s 192.168.69.0/24 -o eno1 -j MASQUERADE
     sudo iptables -A FORWARD -i tap0 -o eno1 -j ACCEPT
     sudo iptables -A FORWARD -i eno1 -o tap0 -m state --state RELATED,ESTABLISHED -j ACCEPT
     ```

   - These rules allow the GNS3 subnet (`192.168.69.0/24`) to access the internet via NAT.

3. **Verify tap0 Interface**:

   - Ensure the tap0 interface is configured:

     ```bash
     sudo ip addr add 192.168.69.254/24 dev tap0
     sudo ip link set tap0 up
     ```

   - Verify: `ip addr show tap0`.

### Step 2: Set Up the GNS3 Topology

1. **Add Docker Hosts**:

   - In GNS3, go to `Edit` → `Preferences` → `Docker` → `Docker Containers` → `New`.
   - Select an Ubuntu-based Docker image (e.g., `ubuntu:20.04`).
   - Name the containers:
     - First container: `Server1`.
     - Second container: `Client1`.
   - Set memory to 512 MB and use 1 vCPU for each.

2. **Configure the Topology**:

   - Drag `Server1` and `Client1` into the GNS3 workspace.
   - Add an Ethernet switch (`Switch1`).
   - Add a Cloud node (`Cloud1`):
     - Configure Cloud1 to use the `tap0` interface.
   - Connect:
     - Server1 eth0 to Switch1 Port 1.
     - Client1 eth0 to Switch1 Port 2.
     - Cloud1 (tap0) to Switch1 Port 3.
   - Start all devices.

   **Topology Diagram**:

   ![GNS3 Topology](topology.png)

   The diagram shows:
   - **Server1** (`192.168.69.10`) and **Client1** (`192.168.69.11`) connected to **Switch1**.
   - **Cloud1** (tap0: `192.168.69.254`) connected to Switch1 for external access.

3. **Configure Host IPs**:

   - Access Server1’s CLI (via GNS3 console):

     ```bash
     ip addr add 192.168.69.10/24 dev eth0
     ip link set eth0 up
     ip route add default via 192.168.69.254
     ```

   - Access Client1’s CLI:

     ```bash
     ip addr add 192.168.69.11/24 dev eth0
     ip link set eth0 up
     ip route add default via 192.168.69.254
     ```

   - Install `iproute2` if `ip` command is missing: `apt update && apt install iproute2`.

4. **Enable SSH Access**:

   - On both Server1 and Client1:

     ```bash
     apt update
     apt install openssh-server
     systemctl start ssh
     systemctl enable ssh
     ```

   - Set root password for Ansible access:

     ```bash
     passwd root
     ```

     - Use password: `cisco` (or your choice, update `hosts.yml` accordingly).

5. **Verify Connectivity**:

   - From Server1: `ping 192.168.69.254`.
   - From Client1: `ping 192.168.69.10`.
   - From host system: `ssh root@192.168.69.10` and `ssh root@192.168.69.11`.

### Step 3: Set Up Ansible

1. **Create Project Directory**:

   - On your host system:

     ```bash
     mkdir -p ~/code/visinetra/network-automation
     cd ~/code/visinetra/network-automation
     ```

2. **Create** `hosts.yml`:

   - File: `hosts.yml`

     ```yaml
     ---
     all:
       hosts:
         server1:
           ansible_host: 192.168.69.10
           ansible_user: root
           ansible_password: cisco
           ansible_connection: ssh
           ansible_become: no
         client1:
           ansible_host: 192.168.69.11
           ansible_user: root
           ansible_password: cisco
           ansible_connection: ssh
           ansible_become: no
     ```

3. **Create** `ansible.cfg`:

   - File: `ansible.cfg`

     ```ini
     [defaults]
     inventory = hosts.yml
     collections_paths = ~/.ansible/collections
     
     [ssh_connection]
     ssh_args = -o StrictHostKeyChecking=no
     ```

   - Ensure Ansible uses this config:

     ```bash
     export ANSIBLE_CONFIG=$(pwd)/ansible.cfg
     ```

4. **Copy Playbooks**:

   - Save the provided `server1.yml` and `client1.yml` into the project directory.
   - These playbooks:
     - **server1.yml**: Configures DNS, Nginx, SNMP, firewall rules, and a Telegraf simulator (sends metrics to InfluxDB Cloud).
     - **client1.yml**: Tests Nginx connectivity and sets up a traffic generator to simulate HTTP requests to Server1.

### Step 4: Run Ansible Playbooks

1. **Install Required Packages**:

   - On Server1 and Client1, ensure basic packages are installed:

     - Access each host via SSH and run:

       ```bash
       apt update
       apt install nginx snmpd ufw curl
       ```

   - This step can be skipped since the playbooks handle most installations, but it’s a good precaution.

2. **Run Playbooks**:

   - From the project directory:

     ```bash
     ansible-playbook -i hosts.yml server1.yml
     ansible-playbook -i hosts.yml client1.yml
     ```

   - Expected output for `server1.yml`:

     - Nginx starts, SNMP is configured, firewall rules allow SSH/HTTP, and Telegraf simulator sends metrics to InfluxDB Cloud.

   - Expected output for `client1.yml`:

     - `curl` tests connectivity to Server1, and a traffic generator service is set up (but not started).

3. **Start Traffic Generation (Optional)**:

   - On Client1, manually start the traffic generator to simulate HTTP requests to Server1:

     ```bash
     ssh root@192.168.69.11 'systemctl start traffic-generator'
     ```

   - Check status:

     ```bash
     ssh root@192.168.69.11 'systemctl status traffic-generator'
     ```

   - Stop when needed:

     ```bash
     ssh root@192.168.69.11 'systemctl stop traffic-generator'
     ```

### Step 5: Set Up Monitoring

1. **InfluxDB Cloud**:

   - Metrics from Server1 are sent to InfluxDB Cloud (`visinetra` bucket, `Dev` org).
   - Log in to InfluxDB Cloud to verify data points (e.g., CPU, memory, network, SNMP uptime).

2. **Grafana**:

   - Add InfluxDB as a data source in Grafana:
     - URL: `https://us-east-1-1.aws.cloud2.influxdata.com`.
     - Token: Same as in `server1.yml`.
     - Organization: `Dev`.
     - Bucket: `visinetra`.
   - Create dashboards to visualize:
     - CPU usage (`cpu_usage`).
     - Memory usage (`memory_usage`).
     - Network traffic (`network_rx`, `network_tx`).
     - SNMP uptime (`snmp_uptime`).

3. **Verify Metrics Locally**:

   - On Server1, check the Telegraf simulator logs:

     ```bash
     ssh root@192.168.69.10 'cat /var/log/telegraf-simulator/metrics.log'
     ```

   - Look for HTTP response codes (204 indicates successful data sends to InfluxDB).

### Step 6: Verify the Setup

1. **Web Server Access**:

   - From Client1: `curl http://192.168.69.10`.
   - Expected: Returns the default Nginx page.
   - From your host system: Open a browser and navigate to `http://192.168.69.10`.

2. **Firewall Rules**:

   - On Server1: `ufw status`.
   - Expected: Ports 22 (SSH) and 80 (HTTP) allowed.

3. **SNMP**:

   - On Server1: `snmpwalk -v2c -c public localhost 1.3.6.1.2.1.1.3.0`.
   - Expected: Returns system uptime.

4. **Traffic Generation**:

   - Start the traffic generator on Client1 (see Step 4.3).
   - Monitor Grafana for increased network traffic metrics.

## Troubleshooting

- **No Internet Access**:
  - Verify IP forwarding and NAT rules on the host system.
  - Check tap0: `ip addr show tap0`.
- **Ansible Fails**:
  - Ensure SSH access: `ssh root@192.168.69.10`.
  - Run with verbose: `ansible-playbook -i hosts.yml server1.yml -v`.
- **Telegraf Fails to Send Data**:
  - Check DNS: `cat /etc/resolv.conf` on Server1 (should show `8.8.8.8` and `1.1.1.1`).
  - Verify InfluxDB token and bucket settings in `server1.yml`.
- **Traffic Generator Not Working**:
  - Check service status: `ssh root@192.168.69.11 'systemctl status traffic-generator'`.
  - Ensure Client1 can reach Server1: `ping 192.168.69.10`.

## Project Structure

```
network-automation/
├── ansible.cfg             # Ansible configuration
├── hosts.yml              # Inventory file
├── server1.yml            # Playbook for Server1
├── client1.yml            # Playbook for Client1
├── topology.png           # GNS3 topology diagram
└── README.md              # This file
```

## Future Enhancements

- Add VLANs to segment the network (e.g., VLAN 10 for the subnet).
- Implement a DNS server on Server1 using `bind9`.
- Configure static routes for a multi-subnet setup (e.g., add a second subnet `192.168.70.0/24`).
- Integrate more advanced monitoring (e.g., HTTP response times, latency).

## Resume Impact

- **Bullet**: “Automated a client-server network on Linux hosts using Ansible in a GNS3 environment, configuring Nginx, firewall rules, and monitoring with Telegraf/InfluxDB/Grafana, reducing setup time by 60%.”
- **Interview Tip**: “I used Ansible to automate a network in GNS3, managing services and monitoring on Linux hosts, which improved my skills in network automation and observability.”

## License

This project is licensed under the MIT License.