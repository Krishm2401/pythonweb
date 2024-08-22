#                                                                        BIGID PROJECT                                                    #
![image](https://github.com/user-attachments/assets/2d7f3e08-b82f-4890-b3fe-09e6ba72a1b2)

![1](https://github.com/user-attachments/assets/42b0815a-97d6-43dc-bd28-45e2504f0882)
			

## SCRIPTS USING:

## REQUIREMENTS.TXT	
```bash
 click==8.0.3
 
 colorama==0.4.4
 
 Flask==2.0.2
 
 itsdangerous==2.0.1
 
 Jinja2==3.0.3
 
 MarkupSafe==2.0.1
 
 Werkzeug==2.0.2
```	

# Dockerfile:
```bash

FROM python:3.10.0-alpine3.15

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY src src

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=30s --start-period=30s --retries=5 \
            CMD curl -f http://localhost:5000/health || exit 1

ENTRYPOINT ["python", "./src/app.py"]
```

# Chart.yaml FILE
```bash
apiVersion: v2
name: webapp
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
 
```

# Values.yaml FILE:
```bash
replicaCount: 2
image:
  repository: krishm2401/bigidhub2
  pullPolicy: Always
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

service:
  type: NodePort
  port: 80
  targetPort: 5000

ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi


nodeSelector: {}

tolerations: []

affinity: {}
```

# deployment.yaml FILE:
```bash                     
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
```
#   service.yaml
```bash
apiVersion: v1
kind: Service
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "webapp.selectorLabels" . | nindent 4 }}
```
# Declartive pipeline :
```bash
pipeline {
    agent any
    stages {
        stage ('SCM checkout') {
            steps {
                script {
                    git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Krishm2401/pythonweb.git'
                }
            }
        }
        stage('Set up Python') {
            steps {
                script {
                    def pythonVersion = '3'
                    sh "python3 -m pip install --upgrade pip"
                    sh "python3 -m pip install -r requirements.txt"
                }
            }
        }
        stage ('SonarQube Code analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonarscanner4'
                    withSonarQubeEnv('sonar-pro') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=big-python"
                    }
                }
            }
        }
        stage('Docker Build Images') {
            steps {
                script {
                    sh 'docker build -t krishm2401/bigidhub2:v1 .'
                    sh 'docker images'
                }
            }
        }
        stage('Docker Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerPass', usernameVariable: 'dockerUser', passwordVariable: 'dockerPassword')]) {
                        sh "docker login -u ${dockerUser} -p ${dockerPassword}"
                        sh 'docker push krishm2401/bigidhub2:v1'
                        sh 'trivy image krishm2401/bigidhub2:v1 > scanning.txt'
                        // sh 'trivy fs --format table -o trivy_report.html'
                    }
                }
            }
        }
        stage('Deploy on k8s') {
            steps {
                script {
                    withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'eks.k8s.local', contextName: '', credentialsId: 'kubernetes', namespace: 'webapps', serverUrl: 'api-eks-k8s-local-jj1tc0-9ca525dbfa608d5d.elb.ap-south-1.amazonaws.com']]) {
                        sh 'kubectl create secret generic helm --from-file=.dockerconfigjson=/opt/docker/config.json --type kubernetes.io/dockerconfigjson --dry-run=client -oyaml > secret.yaml'
                        sh 'helm package ./webapp'
                        sh 'helm list -n webapps -q | xargs -L1 helm uninstall -n webapps'
                        sh 'helm install web98 ./webapp-0.1.0.tgz'
                        sh 'helm ls'
                        sh 'kubectl get pods -o wide'
                        sh 'kubectl get svc'
                    }
                }
            }
        }
    }
    post { 
    always { 
        script { 
            def jobName = env.JOB_NAME 
            def buildNumber = env.BUILD_NUMBER 
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN' 
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red' 
                             def body = """<html> 
                            <body> 
                                <div style="border: 4px solid ${bannerColor}; padding: 10px;"> 
                                    <h2>${jobName} - Build ${buildNumber}</h2> 
                                    <div style="background-color: ${bannerColor}; padding: 10px;"> 
                                        <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3> 
                                    </div> 
                                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p> 
                                </div> 
                            </body> 
                        </html>""" 
            emailext ( 
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}", 
                body: body, 
                to: 'recipent-bharathanavanee@gmail.com', 
                from: 'bharathanavanee@gmail.com', 
                replyTo: 'bharathanavanee@gmail.com', 
                mimeType: 'text/html', 
                attachmentsPattern: 'scanning.txt' 
            ) 
        } 
    } 
}
}
```
# PROCEDURE FOR CICD PIPELINE:
STAGE1:
GIT SCM
Stage2
After installing kops nd kubectl
Step 1
```bash
Apt update
Git Installation
Installing Jenkins
Copy the link form website 
Java installation
Java –version
Jenkins installation
Apt-get install Jenkins
Copy ip address of client M/C
Docker installation
Export the cluster name
Export the state file
Kops create cluster
Printenv
```
# Docker compose installation 

Docker-compose up –d cmd

Memory allocation for containers

Memory allocation for sonarcude has to give only then sonercube will worker

Cmd :sysctl vm.max_map_count=524288

Scripted 
```bash
Cd /var/lib/Jenkins/workspaces/
```
Maven –java


Will bulid on Json format 

Package.json

Templates.yaml(deployment.yaml,
Values.chart-version name
helm-
imperactive way
declarative way 2:03


# Step:1
Installation of kops cluster

Printenv -> is used to print the values of all or specified environment variables.

Bydefault git is preinstalled in Ubuntu AMI

Installation of Jenkins in kops M/C
```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
    https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
  Then add a Jenkins apt repository entry: 
  echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null
  ```
---Update your local package index, then finally install Jenkins---
 ```bash
  sudo apt-get update
  sudo apt-get install fontconfig openjdk-17-jre
  sudo apt-get install jenkins
```

systemctl status jenkins to show running status of jenkins.
http://43.205.114.131:8080/
USE cat  for get token to login jenkins 

# Step:2
Create new instances for sonarqube with t2.medium,30gb EBS volume
```bash
apt update
apt install docker.io –yes
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose –version
```

# Write docker-compose.yaml file for sonarqube
```bash
version: "3"
services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    depends_on:
      - db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    ports:
      - "9000:9000"
    restart: always

  db:
    image: postgres:12
    container_name: postgres12
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
    volumes:
      - postgresql:/var/lib/postgresql
      - postgresql_data:/var/lib/postgresql/data
    restart: always

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  postgresql:
  postgresql_data:

```
```bash sonarqube image will be pulled from default docker repo
assign name for the container
depands on :argument
# 7.For sonerqube perquisite is needed for database
By using depands on is for create database, only then to create sonerqube
db:
image:postgres is the database
env 
sonerqube 
username:
pwd:
Sonerqube is going to be created as a container and login o th loginby using userid and pwd.access the volume
Jdbcportno:5432
/opt/sonerqube/
Sonerqube default portno: 9000:9000
Port Forwarding in same port
# 8.Volume has been created for data,extension and logs, postgress .
JDBC-java database connection
```

vi docker-compose.yaml
```bash docker-compose up –d ```


9.cmd: sysctl -w vm.max_map_count=524288  by using this cmd we can see memory status

vm.max_map_count is greater than or equal to 524288dynamic allocating
```bash 10.Providing memory allocation for containers.
Fs.file-max is greater than or equal to 131072
sysctl vm.max_map_count
You can set them dynamically for the current session by running the following commands as root:

The user running SonarQube can open at least 131072 file descriptors
The user running SonarQube can open at least 8192 threads
You can see the values with the following commands:
Sonerqube will run only when proper memory allocation is done.
```
# http://3.110.132.115:9000/ open in browser
jenkins is CI tool
git plugin has to install in jenkins

# 11.git clone https://github.com/Krishm2401/pythonweb.gitto dolwload pythonweb to git
ls
pythonweb from github account  
```bash
git remote add krish https://github.com/Krishm2401/pythonweb.git
git push -u krish main
```
# 12.Create pipeline
Select pipeline project
Configure -> script pipeline
![image](https://github.com/user-attachments/assets/17dc5da9-fa20-43a3-849f-4c8572e39402)


```bash git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Krishm2401/pythonweb.git' ```
![image](https://github.com/user-attachments/assets/90872f48-85d9-4832-8417-674f59fcde1d)


 
# Click build
First job building process completed

```bash
pipeline {
    agent any
    stages {
        stage ('SCM checkout') {
            steps {
                script {
                    git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Krishm2401/pythonweb.git'
                }
            }

```

# Git SCM checkout is successfully completed by build process in jenkins.
13.Installation of plugins
Sonar-scanner.
 Using the GUI: From your Jenkins dashboard navigate to Manage Jenkins > Manage Plugins and select the Available tab. Locate this plugin by searching for sonar.
 
14.Take the token from GIT, SONARQUBE and give it to Jenkins as credential

GIT TOKEN: ghp_xLd75oNg7UD5ksHiMxJcynNTtqzZxnS11FEtc Kind user name and pwd

SONAR TOKEN:sqa_c29590be8ac9bna9f358cb26186b64583b2fd8955secret text

Dashboard->manage jenkins -.Credentials ->system-> Global credentials (unrestricted)

Dokcer->Kind: secret text

Kubernetes ->kind—secret file.

Id:Soner-token 
![image](https://github.com/user-attachments/assets/ff16efc5-34fb-481e-bf7e-6d83be50628c)

# SONAR SETUP IN JENKINS
![image](https://github.com/user-attachments/assets/86c18ab7-62d9-419b-a614-3523d385707e)
![image](https://github.com/user-attachments/assets/64f44b55-cfba-4b8a-8169-6314e148a442)

 
python – nodejs –sonar-scanner
16.Stage2:
```bash

  stage ('SonarQube Code analysis'){
            steps {
                script{
                    def scannerHome = tool 'sonarscanner4';
                    withSonarQubeEnv('sonar-pro') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=big-python"
                    }
                }
            }
        }
```
![image](https://github.com/user-attachments/assets/70e31c88-964b-44f2-9d38-c8bd2d21fe18)
![image](https://github.com/user-attachments/assets/04dc6d7c-6eb1-4312-a2d7-d53b6a8945e6)

Sonerqube is quality gate is passed
Sonerqube will do code quality test which means will check code level issue, code level vulnerability, code duplicates.
## 17.Stage3:

1.Installation of python3 on Ubuntu

2.sudo apt update

3.sudo apt install python3

4.python –v

5.python3 -m pip install --upgrade pip

6.python3 -m pip install -r requirements.txt

```bash

 stage('Set up Python') {
           steps {
               script {
                   def pythonVersion = '3'
                   sh "python3 -m pip install --upgrade pip"
                   sh "python3 -m pip install -r requirements.txt"
               }
           }
       }
```
By using python3 build process has been completed in jenkins pipeline.
![image](https://github.com/user-attachments/assets/c795056e-3f92-4d73-b115-594cb62d2477)
 
18.Stage4:
```bash
FROM python:3.10.0-alpine3.15
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY src src
EXPOSE 5000
HEALTHCHECK --interval=30s --timeout=30s --start-period=30s --retries=5 \
            CMD curl -f http://localhost:5000/health || exit 1
ENTRYPOINT ["python", "./src/app.py"]
```

Create private repository for the project in docker hub.

docker push krishm2401/bigidhub2:v1     
```bash

       stage('Docker Build Images') {
            steps {  
                script {
                    sh 'docker build -t krishm2401/bigidhub2:v1 .'
                    sh 'docker images'
                }
            }
        }

```


Install in docker in jenkins M/C
And also change permission chmod 700 /var/run/docker.sock
# Stage5: By using cache sonerqube code quality test doing python and requirement.txt is running docker image is built 
![image](https://github.com/user-attachments/assets/2098d85b-1798-4bb2-9280-bd85489b55ac)
20.Docker login succeed.

By using Helm chart going to do deployment 

Has to create Helm chart

Chart.yaml(chart

Values.yaml

Templates.yaml(deployment yaml file manifeast file.

Kubernetes configuration file has to integrate with Jenkins
.kube path/conf

HAS TO INSTALL PLUGINS
Kubernetes 
Kubernetes CLI

Kubernetes URL
Kubernetes server certificate key
Created in txt document kube-config has to upload.

 Has to validate it.
```bash
Use test connection in jenkins page.
By using Helm chart going to do deployment 
Has to create Helm chart
Chart.yaml(api version,versioning)
Values.yaml
Templates.yaml(deployment .yaml file services.yaml file.)
Imagepull secrets:
-name:helm
Is used for pull the image from private repo.

Imperactive-by using CLI cmd
Declarative way-file way
```

# Stage6:
```bash

     stage('Deploy on k8s') {
            steps {
                script {
                    withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'eks.k8s.local', contextName: '', credentialsId: 'kubernetes', namespace: 'webapps', serverUrl: 'api-eks-k8s-local-jj1tc0-9ca525dbfa608d5d.elb.ap-south-1.amazonaws.com']]) {
                        sh 'kubectl create secret generic helm --from-file=.dockerconfigjson=/opt/docker/config.json --type kubernetes.io/dockerconfigjson --dry-run=client -oyaml > secret.yaml'
                        // sh 'kubectl apply -f secret.yaml'
                        // sh 'kubectl apply -f deployment.yaml'
                        // sh 'kubectl apply -f service.yaml'
                        sh 'helm package ./webapp'
                        sh 'helm list -n webapps -q | xargs -L1 helm uninstall -n webapps'
                        sh 'helm install web98 ./webapp-0.1.0.tgz'
                        sh 'helm ls'
                        sh 'kubectl get pods -o wide'
                        sh 'kubectl get svc'
                    }
                }
            }
        }
 ```

cd .docker/
config.json

Has to cp config.json to /opt/docker path.
![image](https://github.com/user-attachments/assets/26d9ea73-2b7f-4878-be02-85d65c873078)

chmod 700 /opt/docker/config.json
kubectl get pods
helm ls
# view the running pods with ip address
```bash kubectl get pods –o wide -n webapps ```
#             OUTPUT 

![image](https://github.com/user-attachments/assets/5d5b9288-9c99-4d92-9fa2-d9289b0d2ce7)

# WEBHOOK TRIGER:
# CREATE WEBHOOK FOR AUTO BUILD:
Give the Jenkins url http://43.205.114.131:8080/github-webhook/
Enable pol scm on Jenkins
![image](https://github.com/user-attachments/assets/3451a270-6bd2-4bbc-846f-fff443bc9f3a)
#  ATTACH EMAIL NOTIFICATION:
  ENABLE EMAIL NOTIFICATION
•  Dashboard
•   Manage Jenkins
•  SystemExtended E-mail Notification
And enable security group inbond smtps 465
![image](https://github.com/user-attachments/assets/84f1129a-cf7b-4af5-ab19-a2179e87ad62)
![image](https://github.com/user-attachments/assets/4242ce70-6c0e-4f52-b98d-28d540b46c62)

# Generate app passward on google account manager for mail credential
![image](https://github.com/user-attachments/assets/692c7cc3-1d15-44c5-ab67-bc8aaf53a984)
# Write pipeline code form mail alert
```bash
post { 
    always { 
        script { 
            def jobName = env.JOB_NAME 
            def buildNumber = env.BUILD_NUMBER 
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN' 
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red' 
                         
                         def body = """<html> 
                            <body> 
                                <div style="border: 4px solid ${bannerColor}; padding: 10px;"> 
                                    <h2>${jobName} - Build ${buildNumber}</h2> 
                                    <div style="background-color: ${bannerColor}; padding: 10px;"> 
                                        <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3> 
                                    </div> 
                                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p> 
                                </div> 
                            </body> 
                        </html>""" 
            emailext ( 
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}", 
                body: body, 
                to: 'recipent-bharathanavanee@gmail.com', 
                from: 'bharathanavanee@gmail.com', 
                replyTo: 'bharathanavanee@gmail.com', 
                mimeType: 'text/html', 
                attachmentsPattern: 'scanning.txt' 
            ) 
        } 
    }
```
# If the pipeline get failed Jenkins send a mail to our email:
![image](https://github.com/user-attachments/assets/2745acc3-018d-43a9-b6df-a8802ff71cc2)
![image](https://github.com/user-attachments/assets/bcab2b86-0f9a-42e5-adf9-3a7844a46c8c)

# If the pipeline run successfully it will also send a mail with trivy image scanning report:
![image](https://github.com/user-attachments/assets/0f6182d5-66b8-4f35-956d-885b55bd369a)
 
# Monitoring :
I have used aws cloudwatch for monitor cpu ,metrics logs for my cluster.
Install cloudwatch agent in my main ec2 and configured it 
![image](https://github.com/user-attachments/assets/1e625526-b20b-40dd-b44d-8c993b3c0807)
![image](https://github.com/user-attachments/assets/c3923c21-2197-46ad-b860-42b70fe332ec)

 
Cloud watch alarm:
![image](https://github.com/user-attachments/assets/ff2c7174-9134-4686-9988-22da3f5e0cd2) 

# ISSUES I FACED IN THIS PROJECT:
1:Sonar qube failed:
![image](https://github.com/user-attachments/assets/8053c870-a5ac-45fc-b19c-5af61db3f095)             
2.IMAGE PULL BACK ERROR AND ERRIMAGEPULL
![image](https://github.com/user-attachments/assets/f6c087c7-2500-446b-878e-ab4dcc73dd99)
![image](https://github.com/user-attachments/assets/0773c66c-8c7a-4a94-a52f-0adf6689395a)
3.DEPLOYMENT ERROR ON YAML FILES

4.KEYPAIR ISSUES 

5.permission deny issues on docker.sock

6.code level issues 
      









#                                                                       THANK YOU
