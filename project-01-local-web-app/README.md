
Project 01 — Local Multi-Tier Web Application Stack
Real-World Scenario
In any production environment, a web application is rarely a single service. It's a stack — a load balancer, an application server, a cache, a message queue, and a database — all talking to each other. Before automating or deploying anything to the cloud, a DevOps engineer needs to understand how that stack fits together and how data flows through it end to end.
This project simulates exactly that. A multi-tier Java web application is deployed locally across five virtual machines, each running a dedicated service, mirroring how this would look in a real datacenter or cloud environment.
This project is the baseline for all projects that follow. Everything built here manually becomes the foundation for automation, containerisation, and cloud deployment in later projects.

What This Project Covers

Provisioning multiple VMs locally using Vagrant and VirtualBox
Understanding and manually configuring each layer of a multi-tier application stack
Service-to-service communication across a private network
Database initialisation and user privilege setup
Caching configuration to improve application response time
The full flow of a user request from browser to database and back


Background — CI/CD Context
Before getting into the build, it's worth understanding the broader process this project sits within.
Continuous Integration (CI) is the practice of automatically building and testing code every time a developer commits to a shared repository. Rather than discovering bugs late, the CI process — code, fetch, build, test, notify, feedback — catches defects early before they multiply. Tools, IDEs, build systems, and version control all plug into this pipeline, with tested artifacts saved to repositories on success.
Continuous Delivery (CD) takes that further. Deployment involves server provisioning, dependency management, configuration changes, networking, and artifact deployment. Automating this entire chain is what CD solves. Common tools in this space include:

Configuration management: Ansible, Puppet, Chef
Infrastructure provisioning: Terraform, CloudFormation
CI/CD orchestration: Jenkins, Octopus Deploy

This project is the manual starting point of that journey — understanding the stack by hand before automating it.

Tools Used
ToolPurposeOracle VirtualBoxLocal hypervisor to run the VMsVagrantVM automation and provisioningGit BashCLI for running commandsVisual Studio CodeIDE for editing configs and codevagrant-hostmanagerVagrant plugin — auto-manages /etc/hosts across all VMs

Application Stack Architecture
ServiceRoleVMIP AddressNginxLoad balancer / web serverweb01192.168.56.11Apache TomcatJava application serverapp01192.168.56.12RabbitMQMessage queue (included in stack)rmq01192.168.56.13MemcachedCaching servicemc01192.168.56.14MySQL / MariaDBDatabase serverdb01192.168.56.15
All VMs sit on the same private network: 192.168.56.0/24

Request Flow
User Browser
    │
    ▼
Nginx (web01 - 192.168.56.11)        ← Load balancer, entry point
    │
    ▼
Apache Tomcat (app01 - 192.168.56.12) ← Hosts the Java application
    │
    ├──▶ Memcached (mc01 - 192.168.56.14)  ← Checks cache first
    │         │ (cache miss)
    │         ▼
    ├──▶ MySQL/MariaDB (db01 - 192.168.56.15) ← Verifies login credentials
    │
    └──▶ RabbitMQ (rmq01 - 192.168.56.13)  ← Message queue (included in stack)
Why this flow matters: If a user can reach the website but cannot log in, the fault is almost certainly in the database layer — not Nginx, not Tomcat. Understanding this flow is what allows a DevOps engineer to isolate and diagnose problems quickly.

Prerequisites

JDK 17 or 21
Maven 3.9
MySQL 8
VirtualBox installed locally
Vagrant installed locally
Git Bash


Technologies

Spring MVC, Spring Security, Spring Data JPA
Maven
JSP
Tomcat
MySQL
Memcached
RabbitMQ
Elasticsearch


Source Code
Clone the repository and navigate to the project directory:
bashgit clone https://github.com/jermaine-mkr/devops-build-projects.git
cd devops-build-projects/project-01-local-web-app

Step-by-Step Build
Step 1 — Install Vagrant Host Manager Plugin
bashvagrant plugin install vagrant-hostmanager
This plugin automatically updates the /etc/hosts file on every VM with the hostnames and IPs of all other VMs in the stack. This means VMs can communicate by hostname rather than requiring manual DNS configuration.

Step 2 — Review the Vagrantfile
The Vagrantfile provisions all five VMs simultaneously. Each VM is assigned a hostname, a static private IP, a base box image, and a memory allocation.

Note: Nginx runs on Ubuntu (jammy64). All other services run on CentOS Stream 9. This reflects real-world mixed-OS environments.

rubyVagrant.configure("2") do |config|
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true

  # MySQL Database VM
  config.vm.define "db01" do |db01|
    db01.vm.box = "generic/centos9s"
    db01.vm.hostname = "db01"
    db01.vm.network "private_network", ip: "192.168.56.15"
    db01.vm.provider "virtualbox" do |vb|
      vb.memory = "600"
    end
  end

  # Memcached VM
  config.vm.define "mc01" do |mc01|
    mc01.vm.box = "generic/centos9s"
    mc01.vm.hostname = "mc01"
    mc01.vm.network "private_network", ip: "192.168.56.14"
    mc01.vm.provider "virtualbox" do |vb|
      vb.memory = "600"
    end
  end

  # RabbitMQ VM
  config.vm.define "rmq01" do |rmq01|
    rmq01.vm.box = "generic/centos9s"
    rmq01.vm.hostname = "rmq01"
    rmq01.vm.network "private_network", ip: "192.168.56.13"
    rmq01.vm.provider "virtualbox" do |vb|
      vb.memory = "600"
    end
  end

  # Tomcat / Application VM
  config.vm.define "app01" do |app01|
    app01.vm.box = "generic/centos9s"
    app01.vm.hostname = "app01"
    app01.vm.network "private_network", ip: "192.168.56.12"
    app01.vm.provider "virtualbox" do |vb|
      vb.memory = "800"
    end
  end

  # Nginx VM
  config.vm.define "web01" do |web01|
    web01.vm.box = "ubuntu/jammy64"
    web01.vm.hostname = "web01"
    web01.vm.network "private_network", ip: "192.168.56.11"
    web01.vm.provider "virtualbox" do |vb|
      vb.gui = true
      vb.memory = "800"
    end
  end

end

Step 3 — Bring Up the VMs
bashvagrant up
To verify all VMs are running:
bashvagrant global-status

Step 4 — Service Setup Order
Always bring up back-end services first, then work forward to the front end:

MySQL (db01)
Memcached (mc01)
RabbitMQ (rmq01)
Tomcat (app01)
Nginx (web01)


Step 5 — MySQL / MariaDB Setup (db01)
SSH into the database VM:
bashvagrant ssh db01
sudo -i
Update the system:
bashdnf update -y
Install MariaDB, then start and enable the service:
bashdnf install -y mariadb-server
systemctl start mariadb
systemctl enable mariadb
Log in to MySQL with default credentials:
bashmysql -u root -padmin123
Create the database, assign privileges, and flush:
sqlCREATE DATABASE accounts;
GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'%' IDENTIFIED BY 'admin123';
FLUSH PRIVILEGES;
EXIT;
Import the database dump to initialise the schema and seed data:
bashmysql -u root -padmin123 accounts < /src/main/resources/db_backup.sql

The db_backup.sql file is located at /src/main/resources/db_backup.sql in the source repository.


Step 6 — Memcached Setup (mc01)
SSH into the Memcached VM:
bashvagrant ssh mc01
sudo -i
dnf update -y
Install, start and enable Memcached:
bashdnf install -y memcached
systemctl start memcached
systemctl enable memcached
By default, Memcached only listens on localhost (127.0.0.1). Since the application server (app01) needs to reach it across the private network, update the config to listen on all interfaces:
bashsed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
Start Memcached with explicit port bindings:
bashmemcached -p 11211 -U 11111 -u memcached -d
PortProtocolPurpose11211TCPMemcached client connections11111UDPMemcached UDP interface

Step 7 — RabbitMQ Setup (rmq01)

(Notes in progress — full setup steps to be added)


Step 8 — Tomcat / Application Setup (app01)

(Notes in progress — full setup steps to be added)


Step 9 — Nginx Setup (web01)

(Notes in progress — full setup steps to be added)


Step 10 — Verify from Browser

(To be completed once all services are configured)

Access the application at:
http://192.168.56.11

Database Info

Dump file location: /src/main/resources/db_backup.sql
Import command: mysql -u <user_name> -p accounts < db_backup.sql


Repo Structure
project-01-local-web-app/
├── README.md
├── Vagrantfile
└── src/
    └── main/
        └── resources/
            └── db_backup.sql
