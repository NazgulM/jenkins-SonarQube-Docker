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

