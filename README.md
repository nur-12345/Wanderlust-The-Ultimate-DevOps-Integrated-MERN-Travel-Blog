# Wanderlust-The-Ultimate-DevOps-Integrated-MERN-Travel-Blog
This project demonstrates a complete end-to-end DevOps pipeline for deploying a production-ready MERN stack travel blog application on an EKS (Elastic Kubernetes Service) cluster.

Wanderlust Mega Project End to End Implementation
In this demo, we will see how to deploy an end to end three tier MERN stack application on EKS cluster.

Tech stack used in this project:

GitHub (Code)
Docker (Containerization)
Jenkins (CI)
OWASP (Dependency check)
SonarQube (Quality)
Trivy (Filesystem Scan)
ArgoCD (CD)
Redis (Caching)
AWS EKS (Kubernetes)
Helm (Monitoring using grafana and prometheus)
How pipeline will look after deployment:

CI pipeline to build and push image

CD pipeline to update application version image

ArgoCD application for deployment on EKS image

Important

Below table helps you to navigate to the particular tool installation section fast.
Tech stack	Installation
Jenkins Master	Install and configure Jenkins
eksctl	Install eksctl
Argocd	Install and configure ArgoCD
Jenkins-Worker Setup	Install and configure Jenkins Worker Node
OWASP setup	Install and configure OWASP
SonarQube	Install and configure SonarQube
Email Notification Setup	Email notification setup
Monitoring	Prometheus and grafana setup using helm charts
Clean Up	Clean up
Pre-requisites to implement this project:

Note

This project will be implemented on North California region (us-west-1).
Create 1 Master machine on AWS with 2CPU, 8GB of RAM (t2.large) and 29 GB of storage and install Docker on it.
Open the below ports in security group of master machine and also attach same security group to Jenkins worker node (We will create worker node shortly) image
Note

We are creating this master machine because we will configure Jenkins master, eksctl, EKS cluster creation from here.
Install & Configure Docker by using below command, "NewGrp docker" will refresh the group config hence no need to restart the EC2 machine.

    sudo apt-get update
    sudo apt-get install docker.io -y
    sudo usermod -aG docker ubuntu && newgrp docker
    Install and configure Jenkins (Master machine)
    sudo apt update -y
    sudo apt install fontconfig openjdk-17-jre -y

    sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
    https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
  
    echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null
  
    sudo apt-get update -y
    sudo apt-get install jenkins -y
    
Now, access Jenkins Master on the browser on port 8080 and configure it.
Create EKS Cluster on AWS (Master machine)

IAM user with access keys and secret access keys
AWSCLI should be configured (Setup AWSCLI)

    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    sudo apt install unzip
    unzip awscliv2.zip
    sudo ./aws/install
     aws configure
Install kubectl (Master machine)(Setup kubectl )

    curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin
    kubectl version --short --client
Install eksctl (Master machine) (Setup eksctl)

    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin
    eksctl version
    
Create EKS Cluster (Master machine)

    eksctl create cluster --name=wanderlust \
                          --region=us-east-2 \
                         --version=1.30 \
                         --without-nodegroup
                         
Associate IAM OIDC Provider (Master machine)

    eksctl utils associate-iam-oidc-provider \
     --region us-east-2 \
     --cluster wanderlust \
     --approve
     
Create Nodegroup (Master machine)

    eksctl create nodegroup --cluster=wanderlust \
                     --region=us-east-2 \
                     --name=wanderlust \
                     --node-type=t2.large \
                     --nodes=2 \
                     --nodes-min=2 \
                     --nodes-max=2 \
                     --node-volume-size=29 \
                     --ssh-access \
                     --ssh-public-key=eks-nodegroup-key 
Note

Make sure the ssh-public-key "eks-nodegroup-key is available in your aws account"
Setting up jenkins worker node
Create a new EC2 instance (Jenkins Worker) with 2CPU, 8GB of RAM (t2.large) and 29 GB of storage and install java on it
sudo apt update -y
sudo apt install fontconfig openjdk-17-jre -y
Create an IAM role with administrator access attach it to the jenkins worker node Select Jenkins worker node EC2 instance --> Actions --> Security --> Modify IAM role image

Configure AWSCLI (Setup AWSCLI)

sudo su
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
generate ssh keys (Master machine) to setup jenkins master-slave
ssh-keygen
image

Now move to directory where your ssh keys are generated and copy the content of public key and paste to authorized_keys file of the Jenkins worker node.

Now, go to the jenkins master and navigate to Manage jenkins --> Nodes, and click on Add node

    name: Node
    type: permanent agent
    Number of executors: 2
    Remote root directory
    Labels: Node
    Usage: Only build jobs with label expressions matching this node
    Launch method: Via ssh
    Host: <public-ip-worker-jenkins>
    Credentials: Add --> Kind: ssh username with private key --> ID: Worker --> Description: Worker --> Username: root --> Private key: Enter directly --> Add Private key
Host Key Verification Strategy: Non verifying Verification Strategy
Availability: Keep this agent online as much as possible
And your jenkins worker node is added

    Install docker (Jenkins Worker)
     sudo apt install docker.io -y
     sudo usermod -aG docker ubuntu && newgrp docker
     
Install and configure SonarQube (Master machine)

    docker run -itd --name SonarQube-Server -p 9000:9000 sonarqube:lts-community
    
Install Trivy (Jenkins Worker)

    sudo apt-get install wget apt-transport-https gnupg lsb-release -y
    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
    echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
    sudo apt-get update -y
    sudo apt-get install trivy -y
    
Install and Configure ArgoCD (Master Machine)
Create argocd namespace
kubectl create namespace argocd
Apply argocd manifest

    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
Make sure all pods are running in argocd namespace

    watch kubectl get pods -n argocd
Install argocd CLI

    sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
Provide executable permission

    sudo chmod +x /usr/local/bin/argocd
Check argocd services

    kubectl get svc -n argocd
Change argocd server's service from ClusterIP to NodePort

    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
Confirm service is patched or not

    kubectl get svc -n argocd
Check the port where ArgoCD server is running and expose it on security groups of a worker node image

Access it on browser, click on advance and proceed with

    <public-ip-worker>:<port>

Fetch the initial password of argocd server

    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    Username: admin
    
Now, go to User Info and update your argocd password

Steps to add email notification

Go to your Jenkins Master EC2 instance and allow 465 port number for SMTPS

Now, we need to generate an application password from our gmail account to authenticate with jenkins

Open gmail and go to Manage your Google Account --> Security
Important

Make sure 2 step verification must be on
image

Search for App password and create a app password for jenkins image image
Once, app password is create and go back to jenkins Manage Jenkins --> Credentials to add username and password for email notification image
Go back to Manage Jenkins --> System and search for Extended E-mail Notification image
Scroll down and search for E-mail Notification and setup email notification
Important

Enter your gmail password which we copied recently in password field E-mail Notification --> Advance
image image image

Steps to implement the project:

Go to Jenkins Master and click on Manage Jenkins --> Plugins --> Available plugins install the below plugins:

    OWASP
    SonarQube Scanner
    Docker
    Pipeline: Stage View
    Configure OWASP, move to Manage Jenkins --> Plugins --> Available plugins (Jenkins Worker) image

After OWASP plugin is installed, Now move to Manage jenkins --> Tools (Jenkins Worker) image

Login to SonarQube server and create the credentials for jenkins to integrate with SonarQube

Navigate to Administration --> Security --> Users --> Token 

Now, go to Manage Jenkins --> credentials and add Sonarqube credentials: image

Go to Manage Jenkins --> Tools and search for SonarQube Scanner installations: image

Go to Manage Jenkins --> credentials and add Github credentials to push updated code from the pipeline: image
Note

While adding github credentials add Personal Access Token in the password field.

Go to Manage Jenkins --> System and search for SonarQube installations: image

Now again, Go to Manage Jenkins --> System and search for Global Trusted Pipeline Libraries:</b image image

Login to SonarQube server, go to Administration --> Webhook and click on create image image

Now, go to github repository and under Automations directory update the instance-id field on both the updatefrontendnew.sh updatebackendnew.sh with the k8s worker's instance id 

    Navigate to Manage Jenkins --> credentials and add credentials for docker login to push docker image: image
Create a Wanderlust-CI pipeline 

Create one more pipeline Wanderlust-CD

Provide permission to docker socket so that docker build and push command do not fail (Jenkins Worker)

    chmod 777 /var/run/docker.sock


Go to Master Machine and add our own eks cluster to argocd for application deployment using cli
Login to argoCD from CLI

     argocd login 52.53.156.187:32738 --username admin
Tip

    52.53.156.187:32738 --> This should be your argocd url


Check how many clusters are available in argocd

    argocd cluster list


Get your cluster name

    kubectl config get-contexts


Add your cluster to argocd
argocd cluster add Wanderlust@wanderlust.us-west-1.eksctl.io --name wanderlust-eks-cluster
Tip

Wanderlust@wanderlust.us-west-1.eksctl.io --> This should be your EKS Cluster Name.


Once your cluster is added to argocd, go to argocd console Settings --> Clusters and verify it image
Go to Settings --> Repositories and click on Connect repo image image image
Note

Connection should be successful
Now, go to Applications and click on New App

How to monitor EKS cluster, kubernetes components and workloads using prometheus and grafana via HELM (On Master machine)

Install Helm Chart

    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
    Add Helm Stable Charts for Your Local Client
    helm repo add stable https://charts.helm.sh/stable
    Add Prometheus Helm Repository
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    Create Prometheus Namespace
    kubectl create namespace prometheus
    kubectl get ns
    Install Prometheus using Helm
    helm install stable prometheus-community/kube-prometheus-stack -n prometheus
    Verify prometheus installation
    kubectl get pods -n prometheus
    Check the services file (svc) of the Prometheus
    kubectl get svc -n prometheus
    Expose Prometheus and Grafana to the external world through Node Port
Important

change it from Cluster IP to NodePort after changing make sure you save the file and open the assigned nodeport to the service.

    kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus


Verify service

    kubectl get svc -n prometheus
Now,let’s change the SVC file of the Grafana and expose it to the outer world

     kubectl edit svc stable-grafana -n prometheus


Check grafana service

    kubectl get svc -n prometheus
Get a password for grafana

    kubectl get secret --namespace prometheus stable-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
Note

Username: admin
Now, view the Dashboard in Grafana
Clean Up

Delete eks cluster

    eksctl delete cluster --name=wanderlust --region=us-west-1

