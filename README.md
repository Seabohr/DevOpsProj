Cloud Native Practical DevOps Project 

Description

This is a python flask application to generate users from a database. In order to deploy it successfully, there are configurations and installations required as pre-requisite. Please see below.

AWS infrastructure
1. Create a VPC with a /16 CIDR block.
2. Create a subnet inside the VPC.
3. Create an IGW from the VPC to the subnet, make sure to associate this with the correct subnet.
4. Create a  route table for the internet gateway.
5. add in a route with a target of the local IGW and destination 0.0.0.0/0.
Create the Bastion EC2
6. Place the bastion EC2 inside the VPC and subnet, enabling a public IP.
7. Create a bastion security group with SSH access only from developers' IPs to ensure security.
Create an AWS EC2 Instance for the services
8. Launch an instance using 'Ubuntu Server 18.04 LTS (HVM), SSD Volume Type Amazon Machine Image (AMI)', with an instance type large enough for the application to run, suggestion: T2 medium. 
9. Place the EC2 inside the VPC and subnet, and enable a public IP to be auto-assigned.
10. Create a services security group for this EC2 with:
	a. SSH access allowed on port 22 from the Bastion Host Instance using its private Ip 	address/32 ONLY.
	b. HTTP access on port 80 from anywhere as NGINX will redirect the traffic as a reverse 	proxy to port 5000.
	c. Access from port 8080 from developer's IPs only to access Jenkins service. Allowing 	this 	only from developer's IP is a control measure for increased security.
11. Use agent forwarding to ssh from Bastion Host to Services public IP. 
12. Once in the Services VM: install Docker.
 - sudo apt update
- sudo apt install curl jq  -y
- sudo apt install curl tree -y (optional)
- curl https://get.docker.com | sudo bash 
- sudo usermod -aG docker $(whoami) -# this adds the current user to the docker group 
13. Install Pip which is the package management system used to install Python:
-  sudo apt install python3-pip -y #installs pip3
14. Install Docker Compose
- sudo apt update
- sudo apt install -y curl jq #ensures both jq and curl are installed.
- version=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | jq -r '.tag_name') #sets which version to download
- sudo curl -L "<https://github.com/docker/compose/releases/download/${version}/docker-compose-$(uname -s)-$(uname -m)>" -o /usr/local/bin/docker-compose 
# downloads to /usr/local/bin/docker-compose.
- sudo chmod +x /usr/local/bin/docker-compose.
# make the file executable.
- to check,  docker-compose --version will show you which version you have.
15.  Install Jenkins
- touch jenkins-install - #creates a script file.
- vim jenkins-install #inside this script file, input the script written below.
 
#!/bin/bash
if type apt > /dev/null; then
    pkg_mgr=apt
    java="openjdk-8-jre"
elif type yum /dev/null; then
    pkg_mgr=yum
    java="java"
fi
echo "updating and installing dependencies"
sudo ${pkg_mgr} update
sudo ${pkg_mgr} install -y ${java} wget git > /dev/null
echo "configuring jenkins user"
sudo useradd -m -s /bin/bash jenkins
echo "downloading latest jenkins WAR"
sudo su - jenkins -c "curl -L https://updates.jenkins-ci.org/latest/jenkins.war --output jenkins.war"
echo "setting up jenkins service"
sudo tee /etc/systemd/system/jenkins.service << EOF > /dev/null
[Unit]
Description=Jenkins Server

[Service]
User=jenkins
WorkingDirectory=/home/jenkins
ExecStart=/usr/bin/java -jar /home/jenkins/jenkins.war

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl restart jenkins
sudo su - jenkins << EOF
until [ -f .jenkins/secrets/initialAdminPassword ]; do
    sleep 1
    echo "waiting for initial admin password"
done
until [[ -n "\$(cat  .jenkins/secrets/initialAdminPassword)" ]]; do
    sleep 1
    echo "waiting for initial admin password"
done
echo "initial admin password: \$(cat .jenkins/secrets/initialAdminPassword)"
EOF

- Run bash jenkins-install #runs the script.
- sudo usermod -aG docker jenkins #adds jenkins to the docker group.
- Reboot the Services instance on AWS and SSH back in.
- take your temporary admin jenkins password and navigate to the Services instance's
public IP on port 8080 and enter in the temporary administrator password.
- install suggested plugins. 
- Create first admin user using a secure password.
- Navigate to Manage Jenkins>Manage Credentials> Add Credentials.
- Select secret text as kind and set your description as database password, ensuring the ID is set to DBPW.
- Repeat this, rather setting SECRET as ID and a required password input for the database set up, however this will not need to be reused so can be any string.
- Repeat this once again, with URI as ID and set the password string to mysql+pymysql://root:[your database password]@database:3306/users, changing the your database password obviously - local will default to 'root' as the password but  across networks will default to 'admin', or you can use your own.
- Create a new Jenkins-Pipeline job
- Under build triggers, choose build periodically, we chose once a day with the CRON code
H0***.
- Under advanced project options, select Pipeline from SCM, then under SCM select Git and paste in the repository URL https://github.com/Seabohr/DevOpsProj where prompted.
- Specify main branch in the branch specifier.
- Set the script path to jenkinsfile and click save and then Build Now.

File Manifest - See GitHub Repo:
https://github.com/Seabohr/DevOpsProj 

Developers' Contact information:
Ashleigh Francis - ashleigh.francis@baesystems.com.
Sebastian Hook - seb.hook@baesystems.com.



