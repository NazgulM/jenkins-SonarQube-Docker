# Jenkins CI/CD Pipeline  - SonarQube, Docker, Github Webhooks

I have created the 3 VM on Digital Ocean, 1 for Jenkins, 1 for SonarQube, and 1 for Docker.

Jenkins machines have installed the jenkins, you can see in jenkinsProjects installation process for it.

## Installing SonarQube on SonarQube VM:

```
sudo yum update -y
sudo shutdown -r
```
1. Install necessary utilities required to configure SonarQube Code Review Tool in CentOS 7

```
sudo yum install vim wget curl unzip -y
```

2. Configure SELinux as Permissive
This can be done by running the commands below:
```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

3. Tweak max_map_count and fs.file-max
From Linux Kernel Documentation, this file contains the maximum number of memory map areas a process may have. Memory map areas are used as a side-effect of calling malloc, directly by mmap, mprotect, and madvise, and also when loading shared libraries.

To tweak the settings to befit SonarQube requirements, open “/etc/sysctl.conf” file and add the settings as shown below:
```
sudo tee -a /etc/sysctl.conf<<EOF
vm.max_map_count=262144
fs.file-max=65536
EOF
```

Apply the settings:

```
sudo sysctl --system
```

Create a user for sonar
It is recommended that a separate user is created to run SonarQube. Let us create one as follows:
```
sudo useradd sonar
```
Then set a password for the user
```
sudo passwd sonar
```
Step 2: Install Java 17 on CentOS 7
As it had been mentioned in the introductory section, SonarQube is written in Java and it needs Java installed

```
sudo yum -y update
```

Install wget command line tool that will be used to download Java 17 binary.
You can get the latest Oracle JDK 17 version from the Oracle Java downloads. Download the required package for your architecture.
```
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.rpm
```

Step 3: Install Java 17 on CentOS 7 / RHEL 7
Use yum or rpm commands to install the package.
```
sudo yum -y install ./jdk-17_linux-x64_bin.rpm
```

```
ls -1 /usr/lib/jvm/jdk-17-oracle-x64/
bin
conf
include
jmods
legal
lib
LICENSE
man
README
release
```

Let’s consider a simple Java program that just prints “Hello World” message.

vim  HelloWorld.java
Paste the following contents into the file
```
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```
Compile the Java source code using the java command:
```
$ java HelloWorld.java
Hello, World!
```

Step 3: Install and configure PostgreSQL
In this example we are going to install PostgreSQL server on the same sever SonarQube will reside. You can host it in a different server depending on your needs. To install PostgreSQL on your CentOS 7 Server, follow the steps below to get it up and running real quick

Add PostgreSQL Yum Repository
Add PostgreSQL Yum Repository to your CentOS 7 system by running the shared command below.
```
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

Install PostgreSQL Server and Client packages
After adding PostgreSQL Yum Repository, install PostgreSQL Server/Client packages:
```
sudo yum -y install postgresql14-server postgresql14
```

After installation, initialize the database and enable automatic start
Now that the database packages have been installed, Initialize the database by running the following command
```
sudo /usr/pgsql-14/bin/postgresql-14-setup initdb
```

Then start and enable the service to start on boot
```
sudo systemctl enable --now postgresql-14
```
After you have installed PostgreSQL server, proceed to configure it as follows. Open pg_hba.conf file and change “peer” to “trust” and “idnet” to “md5“.

```
sudo vim /var/lib/pgsql/14/data/pg_hba.conf
##Change this
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
# IPv6 local connections:
host    all             all             ::1/128                 ident

##To this:
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```

Enable remote Access to PostgreSQL
In case your application is on a remote location, then you will need to allow it to access your database as follows:

Edit the file /var/lib/pgsql/14/data/postgresql.conf and set Listen address to your server IP address or “*” for all interfaces.
```
listen_addresses = '*'
```

Then add the following to “pg_hba.conf” file
```
$ sudo vim /var/lib/pgsql/14/data/pg_hba.conf
# Accept from anywhere
host    all             all             0.0.0.0/0            md5

# Or accept from trusted subnet
host    all             all             10.38.87.0/24        md5
```
Restart PostgreSQL service

```
sudo systemctl restart postgresql-14
```
Set PostgreSQL admin user
We will need to change the password for the admin postgres user as shown below:
```
$ sudo su - postgres
-bash-4.2$: psql 
postgres=# alter user postgres with password 'StrongPassword';
ALTER ROLE
postgres=# exit
```
Create a SonarQube user and database
Next, we are going to create a user for SonarQube. Proceed as shown below before exiting your database.

```
-bash-4.2$ createdb sonarqube
-bash-4.2$ psql
postgres=# CREATE USER sonarqube WITH PASSWORD 'StrongPassword';
postgres=# GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonarqube;
postgres=#  \q
```

Step 4: Fetch and install SonarQube
Now we are at the point we have been waiting to arrive at for a long time. We shall download Long Term Release of SonarQube then install in our Server. Proceed as follows to get our SonarQube installed.

Fetch SonarQube LTS Version
You can visit SonarQube Downloads Page to view their various offerings. We shall be downloading the Long Term Release (LTS)

Downloading via command line interface:
```
cd ~/
wget wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.1.69595.zip
unzip sonarqube-*.zip

After that, rename the folder to sonarqube
```
sudo mv sonarqube-*/  /opt/sonarqube
rm  -rf sonarqube*
```

Step 5: Configure SonarQube
Once the files have been extracted to /opt/ directory, it is time to configure the application.
Open “/opt/sonarqube/conf/sonar.properties” file and add database details as shown below. In addition to that, find the lines shared and uncomment them.
```
$ sudo vim /opt/sonarqube/conf/sonar.properties
##Database details
sonar.jdbc.username=sonarqube
sonar.jdbc.password=StrongPassword
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube

##How you will access SonarQube Web UI
sonar.web.host=0.0.0.0
sonar.web.port=9000

##Java options
sonar.web.javaOpts=-server -Xms512m -Xmx512m -XX:+HeapDumpOnOutOfMemoryError
sonar.search.javaOpts=-Xmx512m -Xms512m -XX:MaxDirectMemorySize=256m -XX:+HeapDumpOnOutOfMemoryError

##Also add the following Elasticsearch storage paths
sonar.path.data=/var/sonarqube/data
sonar.path.temp=/var/sonarqube/temp
```

Give SonarQube files ownership to the sonar user we created in Step 1.

```
sudo chown -R sonar:sonar /opt/sonarqube
sudo mkdir -p /var/sonarqube
sudo chown -R sonar:sonar /var/sonarqube
```

In case Java cannot be found in the default location, you will have to specify the binary files for SonarQube to find. You can specify where java is located in the “/opt/sonarqube/conf/wrapper.conf” file. Look for “wrapper.java.command” line and place your Java location beside it thus.

wrapper.java.command=<custom-path-to-java-binary>
Add SonarQube SystemD service file
Finally we are going to ensure that we shall be able to manage our SonarQube application via Systemd so that we can start and stop it like other services in your server

```
$ sudo vim /etc/systemd/system/sonarqube.service
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
LimitNOFILE=65536
LimitNPROC=4096
User=sonar
Group=sonar
Restart=on-failure

[Install]
WantedBy=multi-user.target

```

After editing systemd files, we have to reload them so that they can be read and loaded.

```
sudo systemctl daemon-reload
Then start and enable the service

sudo systemctl start sonarqube.service
sudo systemctl enable sonarqube.service
Check its status if it successfully started and is running.

systemctl status sonarqube.service
```

Step 6: Alter Firewall rules to allow SonarQube Access
At this juncture, sonarqube service should be running. In case you cannot access the web interface, visit the log files located in “/opt/sonarqube/logs” where you will find

