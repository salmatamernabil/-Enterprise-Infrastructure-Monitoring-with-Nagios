#  Enterprise Infrastructure Monitoring with Nagios

## 1. Project Overview

This project implements centralized infrastructure monitoring for the VProfile multi-tier application using **Nagios Core**.

Nagios Core is an open-source monitoring system used to monitor servers, network services, system resources, and application availability.

A dedicated Nagios virtual machine is deployed using Vagrant and VMware Workstation. The monitoring server communicates with the VProfile machines through NRPE (Nagios Remote Plugin Executor) to collect monitoring information.

### Technologies Used

- Vagrant
- VMware Workstation
- CentOS Stream 9
- Ubuntu
- Nagios Core 4.5.14
- NRPE 4.1.2
- Nagios Plugins
- Apache HTTP Server
- PHP
- Linux Administration
- Bash

## 2. Infrastructure Architecture

### Virtual Machines

| Machine | Role | Private IP |
|---|---|---|
| nagios | Central Monitoring Server | 192.168.56.17 |
| web01 | Web Server | 192.168.56.11 |
| app01 | Application Server | 192.168.56.12 |
| rmq01 | RabbitMQ Server | 192.168.56.13 |
| mc01 | Memcached Server | 192.168.56.14 |
| db01 | Database Server | 192.168.56.15 |

### Monitoring Architecture

The Nagios server communicates with the monitored machines using NRPE over TCP port 5666.

Each target machine runs NRPE and Nagios plugins. Nagios executes remote monitoring commands and receives the results.

## 3. Create the Nagios Virtual Machine

Navigate to the VProfile project directory:

```powershell
cd E:\VProfile
```

Validate the Vagrant configuration:

```powershell
vagrant validate
```

Create and start the Nagios machine:

```powershell
vagrant up nagios
```

Check the machine status:

```powershell
vagrant status
```

Connect to the Nagios server:

```powershell
vagrant ssh nagios
```

Verify its IP address:

```bash
ip -4 addr
```

## 4. Update the Operating System

Update the CentOS packages:

```bash
sudo dnf update -y
```

Install EPEL:

```bash
sudo dnf install epel-release -y
```

## 5. Install Nagios Dependencies

Install development tools:

```bash
sudo dnf groupinstall "Development Tools" -y
```

Install the required dependencies:

```bash
sudo dnf install -y \
gcc \
glibc \
glibc-common \
make \
gettext \
automake \
autoconf \
wget \
openssl-devel \
net-snmp \
net-snmp-utils \
httpd \
php \
php-cli \
gd \
gd-devel \
perl \
unzip \
tar
```

## 6. Download Nagios Core

Navigate to the temporary directory:

```bash
cd /tmp
```

Download Nagios Core 4.5.14:

```bash
wget https://github.com/NagiosEnterprises/nagioscore/releases/download/nagios-4.5.14/nagios-4.5.14.tar.gz
```

Extract the downloaded archive:

```bash
tar -xzf nagios-4.5.14.tar.gz
```

Navigate into the extracted directory:

```bash
cd nagios-4.5.14
```

## 7. Compile and Install Nagios Core

Configure the source code:

```bash
./configure --with-command-group=nagcmd
```

Compile Nagios:

```bash
make all
```

Create the Nagios users and groups:

```bash
sudo make install-groups-users
```

Add Apache to the Nagios command group:

```bash
sudo usermod -aG nagcmd apache
```

Install Nagios:

```bash
sudo make install
```

Install the systemd service:

```bash
sudo make install-daemoninit
```

Install command mode:

```bash
sudo make install-commandmode
```

Install the default configuration files:

```bash
sudo make install-config
```

Install the Apache configuration:

```bash
sudo make install-webconf
```

## 8. Configure Apache Authentication

Create the Nagios administrator account:

```bash
sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```

Enter the password when prompted.

Enable Apache:

```bash
sudo systemctl enable --now httpd
```

## 9. Configure the Firewall

Allow HTTP traffic:

```bash
sudo firewall-cmd --permanent --add-service=http
```

Allow Nagios monitoring traffic if required:

```bash
sudo firewall-cmd --permanent --add-port=5666/tcp
```

Reload the firewall:

```bash
sudo firewall-cmd --reload
```

Port 5666 is primarily required on the target machines for incoming NRPE requests. It does not normally need to be opened on the Nagios server for outbound NRPE checks.

## 10. Start Nagios Services

Enable Nagios:

```bash
sudo systemctl enable nagios
```

Start Nagios:

```bash
sudo systemctl start nagios
```

Enable Apache:

```bash
sudo systemctl enable httpd
```

Check Nagios:

```bash
sudo systemctl status nagios
```

Check Apache:

```bash
sudo systemctl status httpd
```

## 11. Install Nagios Plugins and NRPE Client on the Monitoring Server

The Nagios server needs monitoring plugins to execute local and remote checks.

Install the required packages available in the enabled repositories:

```bash
sudo dnf install -y nagios-plugins
```

Package names can vary by repository. Install the required individual plugins if the package above is unavailable.

The NRPE client is used to communicate with remote machines.

The NRPE client version used in this project is:

**NRPE 4.1.2**

The client executable is available at:

```bash
/usr/lib64/nagios/plugins/check_nrpe
```

The executable is also accessible through the Nagios plugin directory:

```bash
/usr/local/nagios/libexec/check_nrpe
```

Verify the executable:

```bash
ls -l /usr/local/nagios/libexec/check_nrpe
```

Verify its version:

```bash
/usr/local/nagios/libexec/check_nrpe --version
```

## 12. Nagios Server Configuration

Nagios configuration files define the monitored hosts, services, commands, contacts, and monitoring behavior.

### 12.1 Important Configuration Files

| Configuration File | Purpose |
|---|---|
| `/usr/local/nagios/etc/nagios.cfg` | Main Nagios configuration |
| `/usr/local/nagios/etc/resource.cfg` | Resource macros and paths |
| `/usr/local/nagios/etc/objects/commands.cfg` | Monitoring command definitions |
| `/usr/local/nagios/etc/objects/contacts.cfg` | Notification contacts |
| `/usr/local/nagios/etc/objects/templates.cfg` | Reusable host and service templates |
| `/usr/local/nagios/etc/objects/timeperiods.cfg` | Monitoring schedules |
| `/usr/local/nagios/etc/objects/localhost.cfg` | Localhost monitoring |
| `/usr/local/nagios/etc/servers/` | Custom host and service definitions |

### 12.2 Configure nagios.cfg

Open the main configuration:

```bash
sudo nano /usr/local/nagios/etc/nagios.cfg
```

Locate the configuration directory setting:

```ini
cfg_dir=/usr/local/nagios/etc/servers
```

Make sure it is enabled and not commented out.

Create the directory if it does not exist:

```bash
sudo mkdir -p /usr/local/nagios/etc/servers
```

Nagios will automatically read configuration files inside this directory.

### 12.3 Configure NRPE Command

Open:

```bash
sudo nano /usr/local/nagios/etc/objects/commands.cfg
```

Add the following command definition:

```cfg
define command {
    command_name    check_nrpe
    command_line    $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}
```

This command allows Nagios to execute NRPE checks on remote machines.

The `$HOSTADDRESS$` macro contains the IP address of the monitored host.

The `$ARG1$` macro contains the remote command name.

### 12.4 Configure Host Definitions

Create a host configuration file:

```bash
sudo nano /usr/local/nagios/etc/servers/vprofile-hosts.cfg
```

Add the following definitions:

```cfg
define host {
    use             linux-server
    host_name       web01
    alias           VProfile Web Server
    address         192.168.56.11
}

define host {
    use             linux-server
    host_name       app01
    alias           VProfile Application Server
    address         192.168.56.12
}

define host {
    use             linux-server
    host_name       rmq01
    alias           VProfile RabbitMQ Server
    address         192.168.56.13
}

define host {
    use             linux-server
    host_name       mc01
    alias           VProfile Memcached Server
    address         192.168.56.14
}

define host {
    use             linux-server
    host_name       db01
    alias           VProfile Database Server
    address         192.168.56.15
}
```

### 12.5 Configure Service Definitions

Create the service configuration:

```bash
sudo nano /usr/local/nagios/etc/servers/vprofile-services.cfg
```

Add CPU Load monitoring:

```cfg
define service {
    use                     generic-service
    host_name               web01,app01,rmq01,mc01,db01
    service_description     CPU Load
    check_command           check_nrpe!check_load
}
```

Add disk monitoring:

```cfg
define service {
    use                     generic-service
    host_name               web01,app01,rmq01,mc01,db01
    service_description     Disk Usage
    check_command           check_nrpe!check_disk
}
```

Add a user monitoring service if `check_users` is configured on the targets:

```cfg
define service {
    use                     generic-service
    host_name               web01,app01,rmq01,mc01,db01
    service_description     Logged In Users
    check_command           check_nrpe!check_users
}
```

Additional application-specific checks can be configured for HTTP, RabbitMQ, Memcached, and database availability.

## 13. Configure the Target Machines (NRPE)

NRPE allows the Nagios server to execute monitoring plugins on remote Linux machines.

The NRPE configuration must be performed on:

- web01
- app01
- rmq01
- mc01
- db01

### 13.1 Install NRPE

For Ubuntu-based target machines:

Update package repositories:

```bash
sudo apt update
```

Install NRPE:

```bash
sudo apt install -y nagios-nrpe-server
```

Install Nagios plugins:

```bash
sudo apt install -y nagios-plugins
```

For CentOS/RHEL-family target machines:

Install EPEL:

```bash
sudo dnf install -y epel-release
```

Install NRPE:

```bash
sudo dnf install -y nrpe
```

Install the required Nagios plugins:

```bash
sudo dnf install -y nagios-plugins-load nagios-plugins-disk nagios-plugins-users
```

Package availability depends on the enabled repositories and their dependencies. Verify package names using:

```bash
dnf search nagios-plugins
```

### 13.2 Locate the NRPE Configuration File

The configuration file is normally located at:

```bash
/etc/nagios/nrpe.cfg
```

Check its existence:

```bash
sudo find /etc -name nrpe.cfg
```

Open the configuration:

```bash
sudo nano /etc/nagios/nrpe.cfg
```

### 13.3 Configure Allowed Hosts

Locate:

```ini
allowed_hosts=
```

Configure the Nagios server IP:

```ini
allowed_hosts=127.0.0.1,192.168.56.17
```

This allows the Nagios server to communicate with the NRPE agent.

**Important:** Do not configure the Nagios IP with an invalid value such as `192.168.56.17::1`. This can cause NRPE to reject the monitoring server with:

```text
Host 192.168.56.17 is not allowed
```

### 13.4 Configure NRPE Monitoring Commands

The NRPE configuration defines the commands that Nagios can execute remotely.

For CentOS/RHEL-family targets, the plugin directory is commonly:

```bash
/usr/lib64/nagios/plugins
```

For Ubuntu targets, the plugin directory is commonly:

```bash
/usr/lib/nagios/plugins
```

Check the actual plugin location:

```bash
find /usr/lib /usr/lib64 -name check_load -o -name check_disk 2>/dev/null
```

Example configuration for CentOS/RHEL-family targets:

```ini
command[check_load]=/usr/lib64/nagios/plugins/check_load -w 2,1.5,1 -c 4,3,2

command[check_disk]=/usr/lib64/nagios/plugins/check_disk -w 20% -c 10% -p /

command[check_users]=/usr/lib64/nagios/plugins/check_users -w 5 -c 10
```

Example configuration for Ubuntu targets:

```ini
command[check_load]=/usr/lib/nagios/plugins/check_load -w 2,1.5,1 -c 4,3,2

command[check_disk]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /

command[check_users]=/usr/lib/nagios/plugins/check_users -w 5 -c 10
```

These thresholds are examples and can be adjusted based on the resources of each machine.

### 13.5 Enable and Start NRPE

For Ubuntu:

```bash
sudo systemctl enable nagios-nrpe-server
sudo systemctl start nagios-nrpe-server
```

For CentOS/RHEL-family machines:

```bash
sudo systemctl enable nrpe
sudo systemctl start nrpe
```

Verify the service:

```bash
sudo systemctl status nrpe
```

Or:

```bash
sudo systemctl status nagios-nrpe-server
```

### 13.6 Verify NRPE Listening Port

NRPE listens on TCP port 5666 by default.

Check the listening port:

```bash
sudo ss -tulpn | grep 5666
```

Expected result should show NRPE listening on port 5666.

## 14. Configure Firewall on Target Machines

The target machines must allow incoming NRPE connections from the Nagios server.

For Ubuntu with UFW:

```bash
sudo ufw allow from 192.168.56.17 to any port 5666 proto tcp
```

Check firewall status:

```bash
sudo ufw status
```

For CentOS/RHEL-family systems with firewalld:

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.17" port port="5666" protocol="tcp" accept'
```

Reload firewall rules:

```bash
sudo firewall-cmd --reload
```

Verify the firewall configuration:

```bash
sudo firewall-cmd --list-all
```

## 15. Test NRPE Communication from Nagios Server

After installing and configuring NRPE on all target machines, perform connectivity tests from the Nagios server.

### 15.1 Test TCP Connectivity

Test each target:

```bash
nc -vz 192.168.56.11 5666
nc -vz 192.168.56.12 5666
nc -vz 192.168.56.13 5666
nc -vz 192.168.56.14 5666
nc -vz 192.168.56.15 5666
```

A successful connection confirms that the NRPE port is reachable.

### 15.2 Test NRPE Agent

Execute:

```bash
/usr/local/nagios/libexec/check_nrpe -H 192.168.56.11
```

Repeat for the other machines.

The command should return the NRPE version if communication is successful.

### 15.3 Test CPU Load

Execute:

```bash
/usr/local/nagios/libexec/check_nrpe -H 192.168.56.11 -c check_load
```

Test the other targets:

```bash
/usr/local/nagios/libexec/check_nrpe -H 192.168.56.12 -c check_load

/usr/local/nagios/libexec/check_nrpe -H 192.168.56.13 -c check_load

/usr/local/nagios/libexec/check_nrpe -H 192.168.56.14 -c check_load

/usr/local/nagios/libexec/check_nrpe -H 192.168.56.15 -c check_load
```

### 15.4 Test Disk Usage

```bash
/usr/local/nagios/libexec/check_nrpe -H 192.168.56.11 -c check_disk
```

Repeat for the remaining machines.

### 15.5 Test Logged-In Users

```bash
/usr/local/nagios/libexec/check_nrpe -H 192.168.56.11 -c check_users
```

These tests verify that the Nagios server can execute remote commands through NRPE.

## 16. Validate Nagios Configuration

After creating or modifying configuration files, validate the configuration before restarting Nagios.

Execute:

```bash
sudo /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
```

A successful validation should report:

```text
Things look okay - No serious problems were detected during the pre-flight check
```

Restart Nagios:

```bash
sudo systemctl restart nagios
```

Check the service:

```bash
sudo systemctl status nagios
```

## 17. Access the Nagios Web Interface

Open a browser and navigate to the Nagios server's reachable HTTP address:

```text
http://<NAGIOS_SERVER_WEB_IP>/nagios
```

The monitoring network IP is:

```text
192.168.56.17
```

Use the VM's host-reachable network address if the private IP is not accessible directly from Windows.

Login credentials:

- Username: `nagiosadmin`
- Password: The password configured using `htpasswd`.

The Nagios dashboard provides:

- Current Network Status
- Hosts
- Services
- Host Groups
- Service Groups
- Monitoring Processes
- Event Logs

## 18. Troubleshooting

### 18.1 NRPE Connection Refused

Possible causes:

- NRPE service is stopped.
- NRPE is not listening on port 5666.
- Firewall is blocking incoming traffic.

Check the service:

```bash
sudo systemctl status nrpe
```

Check the listening port:

```bash
sudo ss -tulpn | grep 5666
```

### 18.2 Host Not Allowed

Example error:

```text
Host 192.168.56.17 is not allowed
```

Check:

```bash
sudo nano /etc/nagios/nrpe.cfg
```

Verify:

```ini
allowed_hosts=127.0.0.1,192.168.56.17
```

Restart NRPE after modifying the configuration.

### 18.3 NRPE Command Not Defined

Example:

```text
NRPE: Command 'check_load' not defined
```

Check the command definition inside:

```bash
sudo nano /etc/nagios/nrpe.cfg
```

Make sure the command name matches the one used in the Nagios service definition.

### 18.4 Plugin Not Found

Check plugin locations:

```bash
ls -lah /usr/lib64/nagios/plugins/
```

Or:

```bash
ls -lah /usr/lib/nagios/plugins/
```

Verify the executable path inside `nrpe.cfg`.

### 18.5 Nagios UNKNOWN Status

Possible causes:

- NRPE communication failure.
- Incorrect plugin path.
- Undefined remote command.
- Incorrect command arguments.
- Plugin execution permissions.

Test the command directly from the Nagios server:

```bash
/usr/local/nagios/libexec/check_nrpe -H 192.168.56.12 -c check_load
```

### 18.6 Check Nagios Logs

View Nagios logs:

```bash
sudo tail -f /usr/local/nagios/var/nagios.log
```

Check NRPE logs:

```bash
sudo journalctl -u nrpe -f
```

For Ubuntu:

```bash
sudo journalctl -u nagios-nrpe-server -f
```



## References

- Nagios Core Documentation: https://www.nagios.org/documentation/
- Nagios Core GitHub: https://github.com/NagiosEnterprises/nagioscore
- Nagios Plugins: https://github.com/nagios-plugins/nagios-plugins
- NRPE GitHub: https://github.com/NagiosEnterprises/nrpe
- Vagrant Documentation: https://developer.hashicorp.com/vagrant/docs
