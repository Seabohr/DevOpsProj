PRESENTATION:
- decided to use peer programming for the entire project
- Created our git hub repos, and added both team members as collaborators
- Performed a push of the dockerfile from the frontend as a test push to the repo
- Created our Jira board and assigned necessary tasks to specific team members and added collaborators

Completed our risk assessment

Created VPC 
Created our subnet
Created our IGW
Created our route table for our internet gateway 
Creating our instance - we opted for T2 medium - as we wanted to avoid the risk of running into problems with this
enabled a public ip to this 
Service server- with a security group not allowin SSH access from outside traffic only within the VPC
Service Security Group
BUG: Cannot get agent forwarding to work on windows - asked Luke and will do a demo later

- We have installed Jenkins on the machine, 
- Edited the dockerfiles for both front and backend
- We are currently creating the Jenkinsfile trying to add in password management - ONGOING BUG : SOLVED
- We also need to try Luke's suggestion for creating our bastion

Day 2

-In the Jenkinsfile, we separated the build into different stages: installation and compose, to assist
in debugging. We expect the 'build-installation' to run without error - NO LONGER - Jenkins is not designed to install prerequisites, this is better configured manually in the terminal

- Originally we planned to install docker in the jenkinsfile as a separate build stage
but we switched tactics as this is not what Jenkins is used for, and rather we can configure our
EC2 to have docker and docker compose already installed ourselves

BUG: ubuntu no longer had access to sudo commands on our EC2 because we had attempted to
give permissions to Jenkins as sudo so that it could install Docker and Docker Compose meaning
Ubuntu lost sudo access

SOLUTION: Built a new EC2 and configured the environment by installing Docker, Docker Compose and Jenkins ourselves 
without changing sudo access.

We want to:
- Use docker swarm as a stretch goal
- Output the test report as an artifact - 02/09 DONE
- go back and address the risk assessment to include more risks. 02/09 started
- Include full installation guide in ReadMe

BIG WEDNESDAY BUG

Big Bug: Jenkins build of test
Solution and explaination: Each sh line is a completely different shell, meaning each line begins from 
the root directory, so we must cd into the correct location and run the test report command in the same shell

Day 3

Secret key is an extra security measure to prevent data attacks on form submissions
Database_URI - set the environment variables in our environment or in Jenkins as credentials

Configuration for VM
- Install Jenkins manually on the vm using the script provided in a bash file
- Install Docker 
- Install Docker-compose

Infrastructure
- VPC with a /16 CIDR block
- igw attached to VPC
- One subnet inside the VPC
- Create a route table and associate it with the subnet
- Edit the routes. target local IGW and destination 0.0.0.0/0 
EC2 Bastion: place this into the VPC, and the relevant subnet and enable public IP
- Create security group bastion-security-group with SSH access only from the developers
- 

EC2 for the services in the VPC and Subnet
- Enable public IP
- Leave everything else as default 
- Create new security group services-security-group with SSH access from the bastion using its private IP/32
and HTTP access from everywhere for port 80 as NGINX will redirect the traffic as a reverse proxy
- 8080 TCP from developers IP's only to access Jenkins service for extra security


- Use agent forwarding to ssh from bastion host to services private ip. This is how we did it
ssh-add PATH_TO_PRIVATE_KEY #adds your private key to agent
ssh-add -L #outputs all keys in agent
ssh -A USER@BASTION_PUBLIC_IP #The -A flag allows agent forwarding
ssh USER@PRIVATE_IP 

Once in, run: For Docker
- sudo apt update
- sudo apt install curl jq  -y
- sudo apt install curl tree -y (optional)
- curl https://get.docker.com | sudo bash  
- sudo usermod -aG docker $(whoami) - adds the current user to the docker group 

Install pip3: 
sudo apt install python3-pip -y #installs pip3

Docker compose
# make sure jq & curl is installed
sudo apt update
sudo apt install -y curl jq
- # set which version to download (latest)
version=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | jq -r '.tag_name')
# download to /usr/local/bin/docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/${version}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# make the file executable
sudo chmod +x /usr/local/bin/docker-compose
- to check, docker-compose --version

Jenkins
- touch jenkins-install - creates a script file where you can input the script below
- vim jenkins-install 

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

- vim jenkins-install
- bash jenkins-install 
- sudo usermod -aG docker jenkins #adds jenkins to the docker group
- Reboot your services instance 
- take your temporary admin jenkins password and navigate to your services instance 
public ip on port 8080 and enter in the temporary administrator password, install suggested 
plugins 
- Create first admin user using a secure password 
- Nav to Manage Jenkins>Manage Credentials> Add Credentials
- Select secret text as kind and set your description as database password, ensuring the ID is set to 
DBPW
- Add another doing the same, SECRET as ID and a required password input for the database set up, however this will not 
need to be reused
- URI as ID and set the password string to mysql+pymysql://root:[your database password]@database:3306/users, changing
the your database password obviously - if its local it defaults to root but if its across networks it defaults to admin, or you can use your own username
- Create a new Jenkins-Pipeline job
- Under build triggers, choose build periodically, we chose once a day with the CRON code 
H0*** 
- Under advanced project options, select Pipeline from SCM, then under SCM select Git and paste in your repository URL where 
prompted
- Specify main branch in the branch specifier
- Set the script path to jenkinsfile and click save and then BUILD NOW 


- include diagrams of aws infrastructure and ci pipeline in readme.md
- configuration
- test reports in there too - screenshots of test reports for both front and backend

Day 4

- Continuity of service- fresh jenkins build required for webpage to run 
- Going to attempt to use Docker Swarm -

Infrastructure for Docker Swarm
- Create three new EC2s 
- t2 small - fine because we will be using load balancing, add to the VPC AND SUBNET AND enable public ip
Manager Security group SSH access from the bastion
- HTTP access from anywhere 
- 8080 for Jenkins but from developer IPs only
- 2377 - for docker swarm communication
- 7946 TCP UDP - for container network MAYBE
- 4789 - UDP - for container ingress network  MAYBE

Security group for the docker swarm worker nodes
- ?? please may i see the security group for the swarm worker nodes

Configuration for swarm one manager ec2 and two worker ec2
- SSH into the Docker Manager through its private IP as it is still using the bastion
- run sudo apt update
- INSTALL DOCKER - see above instructions
- INSTALL DOCKER COMPOSE - see above instructions 
DO THIS ON THE  MANAGER, AND BOTH WORKER NODES

- Install Jenkins only on the Manager node, and the security group has allowed access from 
developer IP only (8080)

- All nodes install pip - sudo apt install python3-pip -y #installs pip

For pushinig the images to dockerhub
- we had to rebuild them using docker build -t username: image name:version

- After Jenkins reboot but before configuring jenkins, on the Manager node, initiate the swarm 
by doing docker swarm init
- Use the docker swarm join token to join the worker nodes to the swarm 
- check with docker node ls, you should have 2 with nothing under manager and one with leader
status 

- we are using NGINX both as a reverse proxy and a load balancer 
- part of the http section of the nginx.conf we added upstream frontend then 


