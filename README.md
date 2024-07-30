# SonarQube Installation Guide for RHEL

This guide provides step-by-step instructions for installing SonarQube on Red Hat Enterprise Linux (RHEL).

## Table of Contents
1. [Install PostgreSQL](#install-postgresql)
2. [Create Database for SonarQube](#create-database-for-sonarqube)
3. [Install Java 17](#install-java-17)
4. [Increase System Limits](#increase-system-limits)
5. [Install SonarQube](#install-sonarqube)
6. [Access SonarQube UI](#access-sonarqube-ui)
7. [Troubleshooting](#troubleshooting)

## Install PostgreSQL

1. Add the PostgreSQL Repository:
   ```bash
   sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-$(rpm -E %{rhel})-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   ```

2. Disable Red Hat's AppStream PostgreSQL:
   ```bash
   sudo dnf -qy module disable postgresql
   ```

3. Install PostgreSQL:
   ```bash
   sudo dnf install -y postgresql15-server
   ```

4. Initialize the Database and Enable Automatic Start:
   ```bash
   sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
   sudo systemctl enable --now postgresql-15
   ```

5. Check PostgreSQL status:
   ```bash
   sudo systemctl status postgresql-15
   ```

## Create Database for SonarQube

1. Set PostgreSQL user password:
   ```bash
   sudo passwd postgres
   ```

2. Switch to PostgreSQL user:
   ```bash
   su - postgres
   ```

3. Create SonarQube user and database:
   ```bash
   createuser sonar
   psql
   ```

   In the PostgreSQL shell:
   ```sql
   ALTER USER sonar WITH ENCRYPTED PASSWORD 'sonar';
   CREATE DATABASE sonarqube OWNER sonar;
   GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
   \q
   ```

4. Exit PostgreSQL user shell:
   ```bash
   exit
   ```

## Install Java 17

1. Add Adoptium Repository:
   ```bash
   sudo rpm --import https://packages.adoptium.net/artifactory/api/gpg/key/public
   sudo tee /etc/yum.repos.d/adoptium.repo <<REPO
   [Adoptium]
   name=Adoptium
   baseurl=https://packages.adoptium.net/artifactory/rpm/centos/\$releasever/\$basearch
   enabled=1
   gpgcheck=1
   gpgkey=https://packages.adoptium.net/artifactory/api/gpg/key/public
   REPO
   ```

2. Install Java 17:
   ```bash
   sudo yum install temurin-17-jdk
   ```

3. Configure Java Alternatives:
   ```bash
   sudo dnf install java-17-openjdk
   java --version
   ```

   If there's a problem:
   ```bash
   sudo rm -f /etc/yum.repos.d/adoptium.repo
   sudo dnf clean all
   sudo dnf install java-17-openjdk
   ```

## Increase System Limits

1. Edit `/etc/security/limits.conf`:
   ```bash
   sudo vim /etc/security/limits.conf
   ```
   Add the following lines at the bottom of the file:
   ```
   sonarqube   -   nofile   65536
   sonarqube   -   nproc    4096
   ```

2. Edit `/etc/sysctl.conf`:
   ```bash
   sudo vim /etc/sysctl.conf
   ```
   Add:
   ```
   vm.max_map_count = 262144
   ```

3. Apply changes:
   ```bash
   sudo sysctl -p
   sudo reboot
   ```

## Install SonarQube

1. Download and extract SonarQube:
   ```bash
   sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip -P /tmp
   sudo yum install unzip
   sudo unzip /tmp/sonarqube-9.9.0.65466.zip -d /opt
   sudo mv /opt/sonarqube-9.9.0.65466 /opt/sonarqube
   ```

2. Create SonarQube user:
   ```bash
   sudo groupadd sonar
   sudo useradd -c "User to run SonarQube" -d /opt/sonarqube -g sonar sonar
   sudo chown sonar:sonar /opt/sonarqube -R
   ```

3. Update SonarQube properties:
   ```bash
   sudo vim /opt/sonarqube/conf/sonar.properties
   ```
   Modify:
   ```
   sonar.jdbc.username=sonar
   sonar.jdbc.password=sonar
   sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
   ```

4. Create SonarQube service file to manage SonarQube as a service:
   ```bash
   sudo vim /etc/systemd/system/sonar.service
   ```
   Paste the below into the file:
   ```ini
   [Unit]
   Description=SonarQube service
   After=syslog.target network.target

   [Service]
   Type=forking

   ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
   ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

   User=sonar
   Group=sonar
   Restart=always

   LimitNOFILE=65536
   LimitNPROC=4096

   [Install]
   WantedBy=multi-user.target
   ```

5. Start SonarQube:
   ```bash
   sudo systemctl start sonar
   sudo systemctl enable sonar
   sudo systemctl status sonar
   sudo tail -f /opt/sonarqube/logs/sonar.log
   ```

## Access SonarQube UI

Access via: `http://<IP>:9000`

On local: `curl http://localhost:9000`


## Result:

![Ekran görüntüsü 2024-07-22 181144](https://github.com/user-attachments/assets/488e500b-7083-4324-ab14-9eb2d1515aa4)


## Troubleshooting

1. Check Firewall Settings:
Make sure that port 9000 is open on your firewall. You can open it with:
   ```bash
   sudo firewall-cmd --permanent --add-port=9000/tcp
   sudo firewall-cmd --reload
   ```

3. Review SonarQube Logs:
   ```bash
   sudo tail -f /opt/sonarqube/logs/sonar.log
   ```
