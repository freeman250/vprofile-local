# VProfile Project — Local / On-Premises Setup

> 🎓 **Guided Learning Project** — This is a hands-on walkthrough project completed as part of my DevOps learning journey. I followed along with the course by [hkhcoder](https://github.com/hkhcoder/vprofile-project), provisioning and configuring each service myself to build a solid understanding of multi-tier application infrastructure from the ground up.

A multi-tier Java web application stack provisioned locally using Vagrant and VirtualBox. This project demonstrates manual, on-premises infrastructure setup across five services running in separate VMs on your personal machine.

---

## 📐 Architecture Overview

```
User/Browser
     │
     ▼
[ Nginx VM ]  ← Load Balancer / Reverse Proxy (Port 80)
     │
     ▼
[ Tomcat VM ]  ← Java Application Server (Port 8080)
     │
     ├──► [ Memcache VM ]  ← DB Caching (Port 11211)
     ├──► [ RabbitMQ VM ]  ← Message Broker (Port 5672)
     └──► [ MySQL VM ]     ← Relational Database (Port 3306)
```

---

## 🧰 Prerequisites

| Tool | Purpose |
|------|---------|
| [Oracle VirtualBox](https://www.virtualbox.org/) | Hypervisor to run VMs |
| [Vagrant](https://www.vagrantup.com/) | VM provisioning and management |
| [Git Bash](https://git-scm.com/) | Terminal (Windows users) |

Install the Vagrant host manager plugin:
```bash
vagrant plugin install vagrant-hostmanager
```

---

## 🖥️ Virtual Machines

| VM Name | Hostname | Role | OS |
|---------|----------|------|----|
| db01 | db01 | MySQL Database | CentOS/AlmaLinux |
| mc01 | mc01 | Memcache | CentOS/AlmaLinux |
| rmq01 | rmq01 | RabbitMQ | CentOS/AlmaLinux |
| app01 | app01 | Tomcat App Server | CentOS/AlmaLinux |
| web01 | web01 | Nginx Web Server | Ubuntu |

---

## 🚀 Getting Started

### 1. Clone and navigate to the repo

```bash
git clone https://github.com/YOUR_USERNAME/vprofile-local.git
cd vprofile-local
git switch local
cd vagrant/Manual_provisioning
```

### 2. Bring up all VMs

```bash
vagrant up
```

> ⚠️ This may take several minutes. If it stops mid-way, re-run `vagrant up`.

All VM hostnames and `/etc/hosts` entries are automatically managed by the hostmanager plugin.

---

## ⚙️ Provisioning Order

Services **must** be configured in this order due to dependencies:

```
1. MySQL      (Database)
2. Memcache   (DB Caching)
3. RabbitMQ   (Message Broker)
4. Tomcat     (Application Server)
5. Nginx      (Web Server / Reverse Proxy)
```

---

## 🛠️ Service Setup Details

### 1. MySQL (`db01`)

```bash
vagrant ssh db01
dnf update -y
dnf install epel-release git mariadb-server -y
systemctl start mariadb && systemctl enable mariadb
mysql_secure_installation   # set root password: admin123
```

Create the database and user:
```sql
CREATE DATABASE accounts;
GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'%' IDENTIFIED BY 'admin123';
FLUSH PRIVILEGES;
```

Initialise schema from source:
```bash
cd /tmp
git clone -b local https://github.com/hkhcoder/vprofile-project.git
mysql -u root -padmin123 accounts < vprofile-project/src/main/resources/db_backup.sql
```

Open firewall port:
```bash
firewall-cmd --zone=public --add-port=3306/tcp --permanent && firewall-cmd --reload
```

---

### 2. Memcache (`mc01`)

```bash
vagrant ssh mc01
dnf install epel-release memcached -y
systemctl start memcached && systemctl enable memcached
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
systemctl restart memcached
firewall-cmd --add-port=11211/tcp --runtime-to-permanent
```

---

### 3. RabbitMQ (`rmq01`)

```bash
vagrant ssh rmq01
dnf install epel-release wget -y
dnf -y install centos-release-rabbitmq-38
dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
systemctl enable --now rabbitmq-server

# Create admin user
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"

firewall-cmd --add-port=5672/tcp --runtime-to-permanent
```

---

### 4. Tomcat (`app01`)

```bash
vagrant ssh app01
dnf install epel-release java-17-openjdk java-17-openjdk-devel git wget -y

# Download and extract Tomcat 10
cd /tmp
wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10.1.26.tar.gz
tar xzvf apache-tomcat-10.1.26.tar.gz
useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
cp -r apache-tomcat-10.1.26/* /usr/local/tomcat/
chown -R tomcat.tomcat /usr/local/tomcat
```

Create `/etc/systemd/system/tomcat.service` and reload:
```bash
systemctl daemon-reload
systemctl start tomcat && systemctl enable tomcat
firewall-cmd --zone=public --add-port=8080/tcp --permanent && firewall-cmd --reload
```

**Build and Deploy the Application:**
```bash
# Setup Maven
cd /tmp
wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.zip
unzip apache-maven-3.9.9-bin.zip
cp -r apache-maven-3.9.9 /usr/local/maven3.9

# Clone and configure the app
git clone -b local https://github.com/hkhcoder/vprofile-project.git
cd vprofile-project
vim src/main/resources/application.properties  # update backend hostnames

# Build and deploy
export MAVEN_OPTS="-Xmx512m"
/usr/local/maven3.9/bin/mvn install
systemctl stop tomcat
rm -rf /usr/local/tomcat/webapps/ROOT*
cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
systemctl start tomcat
```

---

### 5. Nginx (`web01`)

```bash
vagrant ssh web01
sudo -i
apt update && apt upgrade -y
apt install nginx -y
```

Create `/etc/nginx/sites-available/vproapp`:
```nginx
upstream vproapp {
    server app01:8080;
}
server {
    listen 80;
    location / {
        proxy_pass http://vproapp;
    }
}
```

Activate and restart:
```bash
rm -rf /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
systemctl restart nginx
```

---

## ✅ Verify the Stack

Once all services are up, open your browser and navigate to:
```
http://web01
```

You should see the VProfile login page.

---

## 🗂️ Project Structure

```
.
├── vagrant/
│   └── Manual_provisioning/
│       └── Vagrantfile
├── src/
│   └── main/
│       └── resources/
│           ├── application.properties
│           └── db_backup.sql
└── README.md
```

---

## 🧹 Teardown

```bash
vagrant halt       # Stop all VMs
vagrant destroy    # Delete all VMs
```

---

## 📚 Key Concepts Demonstrated

- Multi-VM provisioning with Vagrant
- Manual service configuration and systemd management
- Firewall rules with `firewalld`
- Reverse proxy setup with Nginx
- Java application deployment with Tomcat and Maven
- Database initialisation with MariaDB
- Message queuing with RabbitMQ
- In-memory caching with Memcache