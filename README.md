# CI/CD Pipeline EKS cluster

## EC2 instance creation:

Create an EC2 instance with t2.large and setup the security groups.

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled.png)

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%201.png)

The following error occurred while trying to connect to the ec2 instance.

> EC2 Instance Connect is unable to connect to your instance. Ensure your instance network settings are configured correctly for EC2 Instance Connect
> 

This was resolved by setting up the internet gateway. Followed the instructions given in this video to resolve the issue - [https://www.youtube.com/watch?v=BaKUtzY1Mis](https://www.youtube.com/watch?v=BaKUtzY1Mis)

Create an IAM role with Admin access and attach it to the EC2 instance.

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%202.png)

## Install the required tools:

Perform the first update with the code:

```bash
sudo apt update
```

Though we need java installed for this project, we can install maven which shall automatically install Java for us. 

```bash
sudo apt install maven -y
mvn --version
java --version
```

Jenkins Setup:

**Add Repository key to the system**

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
/usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

**Append debian package repo address to the system**

```bash
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
[https://pkg.jenkins.io/debian](https://pkg.jenkins.io/debian) binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
```

**Update Ubuntu package**

```bash
sudo apt update
```

**Install Jenkins**

```bash
sudo apt install jenkins -y
```

Use the public IP address and use port 8080 to open jenkins. Copy the dir path shown in the page and get the admin password.

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%203.png)

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install all the suggested plugins

Now inside the EC2 instance install the following:
Install the AWS CLI to access all AWS services, use the following code:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" 

sudo apt install unzip

sudo unzip awscliv2.zip  

sudo ./aws/install

aws --version
```

Install EKS clusters in the EC2 instance

```bash
curl --silent --location "[https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$](https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$)(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

Move the extracted binary to bin folder

```bash
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

Install kubectl in the EC2 instance

```bash
sudo curl --silent --location -o /usr/local/bin/kubectl [https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl](https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl)

sudo chmod +x /usr/local/bin/kubectl

kubectl version --short --client
```

Change to Jenkins user

```bash
sudo su - jenkins
```

Create EKS cluster with 2 worker nodes:

```bash
eksctl create cluster --name cicd-eks-00 --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --managed --nodes 2
```

## Elastic container Registry & Docker install:

Create an Elastic container Registry

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%204.png)

Install docker in the jenkins EC2 instance:

EC2 instance can be accessed using the SSH in vscode. Run the command shown below in VS code and make sure the .pem file (access key) is the same path where the command is being run.

```bash
ssh -i "jenkins.pem" [ubuntu@ec2-34-205-25-157.compute-1.amazonaws.com](mailto:ubuntu@ec2-34-205-25-157.compute-1.amazonaws.com)
```

Installation of docker

```bash
sudo apt install [docker.io](http://docker.io/) -y
```

Faced an error while trying to install docker:
First change the password of jenkins user by changing to root user and using this command.

```bash
passwd jenkins
```

Then will trying to install docker this error popped up:

> jenkins is not in the sudoers file. This incident will be reported.
> 

This error was resolved by following the steps in this link - [https://stackoverflow.com/questions/47806576/linux-username-is-not-in-the-sudoers-file-this-incident-will-be-reported](https://stackoverflow.com/questions/47806576/linux-username-is-not-in-the-sudoers-file-this-incident-will-be-reported)

Now add Ubuntu user to Docker group, for this first switch user to Ubuntu and then use the command below:

```bash
sudo usermod -aG docker $USER
```

Run all the commands shown below to run the docker service:

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

Since Jenkins will performing Docker builds it is important to add Jenkins user in Docker group. Also reload the daemon files.

```bash
sudo usermod -a -G docker jenkins
sudo systemctl daemon-reload

sudo service docker stop
sudo service docker start
```

Now Log into Jenkins and install some important plugins:

- Docker
- Docker Pipelin
- Kubernetes CLI
- SonarQube Scanner

Maven should be registered in Jenkins:

![The Maven_Home and the version details can be seen by using the command mvn —version.](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%205.png)

The Maven_Home and the version details can be seen by using the command mvn —version.

Create Credentials to connect with the Kubernetes cluster using the kubeconfig.

## Create Jenkins pipeline:

### Stage 1: Checkout

A repo with the source code has been created and the JenkinsFile is also added in the repo. 

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%206.png)

![Since we have separeate Jenkins file in git hub, we are given the repo path to Jenkins and it can read the JenkinsFile.](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%207.png)

Since we have separeate Jenkins file in git hub, we are given the repo path to Jenkins and it can read the JenkinsFile.

![Make sure the script path is correct.](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%208.png)

Make sure the script path is correct.

Using the pipeline syntax option we can generate the syntax required for git checkout: option is checkout: Checkout from version control. This must have the details like the repo URL, and the branch in my case it is main.

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%209.png)

This will successfully complete the 1st stage i n the pipeline which is git checkout.

 

```groovy
pipeline {
  agent  any 
  stages {
        stage('Checkout') {
         steps {
            checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/karthi770/cicd_eks_00.git']])
      }
```

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2010.png)

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2011.png)

### Stage 2: Build Jar file

Following syntax is used to build the Jar file:

```groovy
pipeline {
  agent  any 
 
  tools {
    maven "Apache Maven 3.6.3"
  }
  stages {
        stage('Checkout') {
         steps {
            checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/karthi770/cicd_eks_00.git']])
      }
        }
       stage('Build Jar') {
         steps {
          sh "cd spring-boot-app && mvn clean package"
           sh 'ls -ltr'
         }
      }
```

### Stage 3: Static code analysis

For static code analysis Sonarqube is being used in this project. Following are the installation syntax of SonarQube:

```bash
apt install unzip
adduser sonarqube
sudo su - sonarqube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
unzip *
chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```

Inside Sonarqube generate a token which can be used inside jenkins inorder to authenticate.

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2012.png)

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2013.png)

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2014.png)

### Stage 4: Build Image

Declare the docker registry:

```groovy
environment {
	registry = "634211996823.dkr.ecr.us-east-1.amazonaws.com/spring-boot"
}
```

Now the next stage is to build the image. For which the syntax given below can be used.

```groovy
stage("Build image"){
        steps{
          script {
            docker.build registry
          }
        }

      }
```

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2015.png)

> COPY failed: file not found in build context or excluded by .dockerignore: stat target/spring-boot-web.jar: file does not exist
> 

I got the above error during the building image, the reason for the error is due to the incorrect path of the target folder and the JAR file. Below is the corrected docker file:

```groovy
FROM adoptopenjdk/openjdk11:alpine-jre

WORKDIR /opt/app

COPY spring-boot-app/target/spring-boot-web.jar app.jar

ENTRYPOINT ["java","-jar","app.jar"]
```

### Stage 5: Push the image in ECR

In order to push the image in ECR the following command shall be used, these push commands can be found in the ECR repo. Select the repo and click on “view push commands”

```groovy
stage("push in ECR"){
        steps{
          sh"aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 634211996823.dkr.ecr.us-east-1.amazonaws.com"
          sh"docker push 634211996823.dkr.ecr.us-east-1.amazonaws.com/spring-boot:latest"
        }
      }
```

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2016.png)

![The docker image has been pushed into the ECR successfully.](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2017.png)

The docker image has been pushed into the ECR successfully.

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2018.png)

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2019.png)

Even though the clusters are up and running, we can get the status from the command given below:

```bash
eksctl get cluster --name cicd-eks-00 --region us-east-01
```

Also the command below gives the number of worker nodes running or EC2 instances running.

```bash
kubectl get nodes
```

### Stage 6: Deployment

Before the deployment we have to create credentials for connecting to Kuberbetes cluster using kubeconfig. Use the command below to view the config file.

```bash
cat /var/lib/jenkins/.kube/config
```

Copy the whole content and paste it in notepad and save it as kubeconfig.

Now go to jenkins > Dashboard > Manage Jenkins > Credentials > System > **Global credentials (unrestricted) > Add credentials**

![Select the kubconfig file form the desktop.](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2020.png)

Select the kubconfig file form the desktop.

To generate the code for deployment, click on pipeline syntax and select “withKubeConfig”

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2021.png)

![Copy the generated code and paste it in the jenkinsfile as the last stage.](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2022.png)

Copy the generated code and paste it in the jenkinsfile as the last stage.

```groovy
stage("K8S Deployment"){
        steps{
          withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'K8S', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
            sh "kubectl apply -f deployment.yml"
}
```

In the deployment.yml change the image name as image URI from the ECR repo.

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2023.png)

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2024.png)

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2025.png)

![Untitled](CI%20CD%20Pipeline%20EKS%20cluster%2024d05a2bfd0c465798004f5450963302/Untitled%2026.png)