##End goal: For each commit on a branch, the code get analysed by sonar and you get a slack msg with the results

#Installation & configuration
Prepare two ec2 instances, one for jenkins and sonarscanner (t2.small) and one for sonarqube (t2.large)
I tried t2.small but for some reason it staled sonarqube pid after jenkins pipeline finished execution, so I had to upgrade

aws security group:
port range
9000
8080
22
443
protocol
TCP
TCP
SSH
HTTPS

#On sonarqube server:

Install java https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-on-ubuntu-18-04
You gonna need, wget unzip
cd /opt/
mkdir sonarqube && cd sonarqube
wget last LTS sonarqube version from here https://www.sonarqube.org/downloads/
Don’t run your server as root, do this.
useradd sonaradmin
passwd sonaradmin
cd back and change permissions chown -R sonaradmin:sonaradmin sonarqube
~
sudo apt update && sudo apt upgrade
sudo apt install postgresql postgresql-contrib
sudo postgresql-setup --initdb
find / -name pg_hba.conf
nano pg_hba.conf, and make it like the screenshot, change IPv4 to host where you want to connect
find / -name postgresql.conf, and change listen_addresses to ‘*’ if you want to connect from anywhere 
~
service postgresql start
#sudo systemctl start postgresql
su - postgres
#sudo -u postgres psql
#sudo su - postgres
\password postgres
create user SONAR with encrypted password ‘SONAR’;
create database sonarqubedb;
grant all privileges on database sonarqubedb to SONAR
\q
~
find / -name sonar.properties
nano and do this...
sonar.jdbc.username=SONAR
sonar.jdbc.password=SONAR
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqubedb
sonar.web.host=0.0.0.0
sonar.web.port=9000
sudo su - sonaradmin
cd /opt/sonarqube/bin/linux-x86-64 
sh ./sonar.sh start
sh ./sonar.sh status
~
#Setting up sonarqube as a service
sudo vi /etc/systemd/system/sonarqube.service
“””
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=simple
User=sonarqube
Group=sonarqube
PermissionsStartOnly=true
ExecStart=/bin/nohup java -Xms32m -Xmx32m -Djava.net.preferIPv4Stack=true -jar /opt/sonarqube/lib/sonar-application-7.6.jar
StandardOutput=syslog
LimitNOFILE=65536
LimitNPROC=8192
TimeoutStartSec=5
Restart=always

[Install]
WantedBy=multi-user.target
“””
sudo systemctl start sonarqube
sudo systemctl enable sonarqube
sudo systemctl status  sonarqube
~
#Troubleshooting Sonarqube

cd /opt/sonarqube/logs
es.log
sonar.log
web.log
Access.log
For latest logs, tail -f access.log

#On jenkins server:

Install java, Maven & Jenkins
Jenkins guide: https://www.jenkins.io/doc/book/installing/linux/
You gonna need, wget unzip
cd /opt/
mkdir sonarscanner && cd sonarscanner
wget your version from here https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/
cd conf
nano sonar-scanner.properties
Change sonar.host.url to your sonarqube server private ip (same vpc)
~
Go to jenkins
Install SonarQube Scanner plugin
Go to Global Tool Configuration
Add SonarQube Scanner
Name: any
SONAR_RUNNER_HOME, path to sonarscanner on server
uncheck install automatically
Go to Configure Systems
SonarQube servers
Name: any
Server url: url
Token from sonar -> administration -> security
Install Slack Notification Plugin
Go to Slack, Install jenkins CI, Add jenkins CI integration, follow the setup guide
In Jenkins > plugin manager, Install Bitbucket Plugin
~
Pipeline configurations

Create new freestyle job,
Specify source code management
Branches to build, refs/remotes/origin/develop, will follow develop branch
In Build, Execute SonarQube Scanner
In Analysis properties, write down sonar configs

sonar.projectKey=ebs-jenkins-sonar
sonar.sources=.
sonar.java.binaries=.

these configs are for multi module java applications, change accordingly
~
In Build Triggers,
choose Build when a change is pushed to BitBucket > for triggering the pipeline for each commit on repo
~
In Post-build Actions,
check the following, Notify Build Start, Notify Success, Notify Every Failure, Notify Back To Normal.
~
Click Advanced settings, and  parse a custom msg for desired stage
~
