
The ultimate ci cd corporate devops pipeline project 

Create seperate servers for each : 
Jenkins master / slave - 1
Sonar qube - 1
Nexus or artifactory - 1
Kubernetes ( master node, slave node1, slave nose 2) - 3



1.)
Install jenkins on jenkins server and configure 
Install docker also. Install trivy.
Give docker user and group permission 
Install plugins :- Sonarqube, docker, docker pipeline, nexus artifict uploader, OWASP, jdk eclipse temurin installer, pipeline Maven integration, config file provider plugin
Configure tools jdk17, jdk21, Sonarqube scanner, maven3, dependency-check 6.5.1, docker
Save Sonarqube token as secret text
Save docker credentials 
Add Sonarqube server in jenkins system
Add plugin - config file provider
Configure nexus credentials for maven-releases and maven-snapshots in config file management 
Install kubectl using snap on jenkins server 
Add jenkins to docker group


2.)
Install docker on Sonarqube server, and run Sonarqube container on port 9000
Create token 



3.)
Install docker on nexus server and run nexus container on port 8081. 
Go inside container and find default admin password 


4.)
Configure url for maven-releases and maven-snapshots in pom.xml in distribution management 
Configure jenkins pipeline 


5.)
Kubernetes setup
Configure 3 ubuntu instances 
Install docker & kubernetes on all 3 instances 
Configure kubernetes cluster on master node
and join other two nodes 
Create webapps namespace 
Create service account name jenkins using yml
Create role (what kind of access this role gives to assigned resource like service account) using yml
Assign role to service account we created 
Create secret token for our service account jenkins
View the secret token
View the config file from kube folder


Configure jenkins pipeline step for docker
Configure kubernetes pipeline step
Add plugins :- kubernetes, kubernetes CLI
Add kubernetes token in jenkins credentials 

