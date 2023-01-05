# Continuous Delivery of Java Web Application

In the previous [project](https://github.com/hacizeynal/Continuous-Integration-Using-Jenkins-Nexus-Sonarqube-Slack)
, we built a Continuous Integration pipeline on Jenkins .In this project we will continue to build Continuous Delivery project ,which means that we will need to deploy code automatically . We will use following tools for the project 

* Jenkins
* Maven
* Checkstyle
* Slack 
* SonaType Nexus
* SonarQube
* Docker
* ECR
* ECS
* AWS CLI

High level overview for the project is described below

[![Screenshot-2022-11-09-at-09-08-59.png](https://i.postimg.cc/pTQQxK6L/Screenshot-2022-11-09-at-09-08-59.png)](https://postimg.cc/47dHQHGD)

## STEPS

[![Screenshot-2022-11-09-at-20-38-21.png](https://i.postimg.cc/MTQ503xG/Screenshot-2022-11-09-at-20-38-21.png)](https://postimg.cc/YGtQM3vc)

[![Screenshot-2022-11-09-at-20-39-27.png](https://i.postimg.cc/sgMhZQ3j/Screenshot-2022-11-09-at-20-39-27.png)](https://postimg.cc/Xr0JRvZm)

### Create IAM User

We will create new IAM user **cicdjenkins** ,this user will access to ECR repository and ECS service.
Jenkins will run AWS CLI from pipeline and this user will execute commands for ECR and ECS.

### Configure Elastic Container Registry

We will create private artifact repository on AWS and upload our generated artifacts from Jenkins pipeline to ECR.

[![Screenshot-2022-11-11-at-20-22-36.png](https://i.postimg.cc/50QWsDgJ/Screenshot-2022-11-11-at-20-22-36.png)](https://postimg.cc/dkFf1HgN)

### Configure Additional Settings on Jenkins

We will configure additional configuration in our previous Jenkins server

* Install necessary plugins (Docker Pipeline ,CloudBees Docker Build and publish,AWS ECR,Pipeline: AWS Steps)
* Store new credentials on it (credentials for cicdjenkins)
* Install Docker Engine

Since we will install correct plugins in Jenkins ,we will see **AWS Credentials** type of credentials in Jenkins.

### Create Docker image

We will use multistage Docker in our project

```
FROM openjdk:8 AS BUILD_IMAGE
RUN apt update && apt install maven -y
RUN git clone -b vp-docker https://github.com/imranvisualpath/vprofile-repo.git
RUN cd vprofile-repo && mvn install

FROM tomcat:8-jre11

RUN rm -rf /usr/local/tomcat/webapps/*

COPY --from=BUILD_IMAGE vprofile-repo/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080
CMD ["catalina.sh", "run"]
```
We will create artifact with maven then copy this artifact to another tomcat server and run it.


### Update Jenkinsfile

We will need to add 3 more variable to in order to upload Container to ECS and following commands in order to upload artifacts to ECR.

```
registryCredentials = "ecr-us-east-1:cicdjenkins"
appRegistry = "866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops"
applicationRegistry = "https://866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops"

```

```
        stage("Build Docker Image"){
            steps{
                script{
                    dockerImage = docker.build(imagename + ":$BUILD_ID" + "_$BUILD_TIMESTAMP","./Docker-files/app/multistage/")
                }
            }
        }
        stage("Upload Docker to ECR"){
            steps{
                script{
                    docker.withRegistry (applicationRegistry,registryCredentials){
                        dockerImage.push("$BUILD_ID")
                        // dockerImage.push("$BUILD_TIMESTAMP")
                        dockerImage.push("latest")
                    }
                }
            }
        }

```
As a result following logs will be generated after successfull build.

```
$ docker login -u AWS -p ******** https://866308211434.dkr.ecr.us-east-1.amazonaws.com
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /var/lib/jenkins/workspace/CI_CD_PIPELINE@tmp/40547163-9e27-4671-8cc4-cded0493e5a7/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[Pipeline] {
[Pipeline] isUnix
[Pipeline] withEnv
[Pipeline] {
[Pipeline] sh
+ docker tag 866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops:36_2022-11-17_13_06 866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops:36
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] isUnix
[Pipeline] withEnv
[Pipeline] {
[Pipeline] sh
+ docker push 866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops:36
The push refers to repository [866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops]
aa2e51f5ab8a: Preparing
f4a670ac65b6: Preparing
9a797133ad85: Waiting
13ec7f04879a: Waiting
76d6ce51a35d: Layer already exists
8b4173f33a28: Layer already exists
0439bc492ca9: Layer already exists
36: digest: sha256:9e3b339ae352612fb1059e1977909a0bcfd4c0e13f2fb750feabf31d9797f1a3 size: 2209
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] isUnix
[Pipeline] withEnv
[Pipeline] {
[Pipeline] sh
+ docker tag 866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops:36_2022-11-17_13_06 866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops:latest
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] isUnix
[Pipeline] withEnv
[Pipeline] {
[Pipeline] sh
+ docker push 866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops:latest
The push refers to repository [866308211434.dkr.ecr.us-east-1.amazonaws.com/zhajili_devops]

```
We can see that **BUILD_ID** and **BUILD_TIMESTAMP** is appended to docker image ,but we are only pushing docker images to the ECR with  **latest** tag and **BUILD_ID** tag.

### Configure AWS ECS

We will deploy our Docker container in the Amazon ECS. Amazon ECS will fetch docker images from Amazon ECR. After successful build of pipeline ,we will promote release to the Production environment.

We will create Tasks in ECS in order to run containers .With Amazon ECS, our containers are defined in a task definition that we use to run an individual task or task within a service.

[![Screenshot-2022-12-11-at-21-39-12.png](https://i.postimg.cc/63zdkNW3/Screenshot-2022-12-11-at-21-39-12.png)](https://postimg.cc/njjjmgQb)

We create service which will use Tasks to create containers. Another very important point is that a Service can be configured to use a load balancer, so that as it creates the Tasks—that is it launches containers defined in the Task Definition—the Service will automatically register the container's EC2 instance with the load balancer. Tasks cannot be configured to use a load balancer, only Services can. Nice article [article](https://www.freecodecamp.org/news/amazon-ecs-terms-and-architecture-807d8c4960fd/) which discusses difference between service and task.

Few verification photos

[![Screenshot-2023-01-05-at-10-13-16.png](https://i.postimg.cc/cCyPnrhJ/Screenshot-2023-01-05-at-10-13-16.png)](https://postimg.cc/TL0k82Z8)

We can see that our task is running and we have load balancer attached via service to the task ,you can see that Launch Type is Fargate which serverless mode of compute part ,we can use EC2 ,but it will be more expensive than Fargate.

[![Screenshot-2023-01-05-at-10-14-48.png](https://i.postimg.cc/MKsdknBj/Screenshot-2023-01-05-at-10-14-48.png)](https://postimg.cc/1fqpw5J9)

### Configure Jenkins for Continuous Delivery - Staging Pipeline

Now we will add few steps to our Jenkinsfile ,so when we will run our pipeline it will fetch docker image from ECR and run it on ECS ,from our Jenkins AWS CLI command will run and it will tell Service to fetch latest docker image from ECR repository ,currently we have following docker image with different tag ,tag ID is taken from BUILD ID.

[![Screenshot-2023-01-05-at-10-27-07.png](https://i.postimg.cc/1thrs6MJ/Screenshot-2023-01-05-at-10-27-07.png)](https://postimg.cc/VSgCFJxC)

We will add 2 new variables to Jenkinsfile and we will create new stage for deploying containers to ECS.

```
    environment{
        ecs_cluster_name = "staging"
        service_name = "STAGING-SERVICE"
    }

    stage("Deploy to ECS Staging"){
    steps{
        withAWS(credentials: "cicdjenkins",region: "us-east-1" ){
        // force to delete old container deploy new one via service
        sh 'aws ecs update-service --cluster ${ecs_cluster_name} --service ${service_name} --force-new-deployment'
        }
    }
```
Let's run the pipeline and see the result


```
+ aws ecs update-service --cluster staging --service STAGING-SERVICE --force-new-deployment
{
    "service": {
        "serviceArn": "arn:aws:ecs:us-east-1:866308211434:service/staging/STAGING-SERVICE",
        "serviceName": "STAGING-SERVICE",
        "clusterArn": "arn:aws:ecs:us-east-1:866308211434:cluster/staging",
        "loadBalancers": [
            {
                "targetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:866308211434:targetgroup/STAGING-TARGET-GROUP/f82a88e72d98f552",
                "containerName": "vprofile",
                "containerPort": 8080
            }
        ],
        "serviceRegistries": [],
        "status": "ACTIVE",
        "desiredCount": 1,
        "runningCount": 1,
        "pendingCount": 0,
        "launchType": "FARGATE",
        "platformVersion": "LATEST",
        "taskDefinition": "arn:aws:ecs:us-east-1:866308211434:task-definition/task_staging:1",
        "deploymentConfiguration": {
            "maximumPercent": 200,
            "minimumHealthyPercent": 100
        },
        "deployments": [
            {
                "id": "ecs-svc/6062390510379847045",
                "status": "PRIMARY",
                "taskDefinition": "arn:aws:ecs:us-east-1:866308211434:task-definition/task_staging:1",
                "desiredCount": 0,
                "pendingCount": 0,
                "runningCount": 0,
                "createdAt": 1672914138.248,
                "updatedAt": 1672914138.248,
                "launchType": "FARGATE",
                "platformVersion": "1.4.0",
                "networkConfiguration": {
                    "awsvpcConfiguration": {
                        "subnets": [
                            "subnet-075a646c201c7b88c",
                            "subnet-07d38a366dfbba035",
                            "subnet-05e222a67edc6403a"
                        ],
                        "securityGroups": [
                            "sg-0a0d2d1fbfea4e1f1"
                        ],
                        "assignPublicIp": "ENABLED"
                    }
                }
            },
            {
                "id": "ecs-svc/1062741110952534332",
                "status": "ACTIVE",
                "taskDefinition": "arn:aws:ecs:us-east-1:866308211434:task-definition/task_staging:1",
                "desiredCount": 1,
                "pendingCount": 0,
                "runningCount": 1,
                "createdAt": 1672909747.903,
                "updatedAt": 1672909881.155,
                "launchType": "FARGATE",
                "platformVersion": "1.4.0",
                "networkConfiguration": {
                    "awsvpcConfiguration": {
                        "subnets": [
                            "subnet-075a646c201c7b88c",
                            "subnet-07d38a366dfbba035",
                            "subnet-05e222a67edc6403a"
                        ],
                        "securityGroups": [
                            "sg-0a0d2d1fbfea4e1f1"
                        ],
                        "assignPublicIp": "ENABLED"
                    }
                }
            }
        ],
        "roleArn": "arn:aws:iam::866308211434:role/aws-service-role/ecs.amazonaws.com/AWSServiceRoleForECS",
        "events": [
            {
                "id": "a638271a-4cbb-43d0-880a-9372f444eb89",
                "createdAt": 1672909881.162,
                "message": "(service STAGING-SERVICE) has reached a steady state."
            },
            {
                "id": "13ab9b88-f7dc-45ff-addc-bd6bea28663b",
                "createdAt": 1672909881.161,
                "message": "(service STAGING-SERVICE) (deployment ecs-svc/1062741110952534332) deployment completed."
            },
            {
                "id": "ac1fc499-0266-4f07-842e-3677b31635c2",
                "createdAt": 1672909783.532,
                "message": "(service STAGING-SERVICE) registered 1 targets in (target-group arn:aws:elasticloadbalancing:us-east-1:866308211434:targetgroup/STAGING-TARGET-GROUP/f82a88e72d98f552)"
            },
            {
                "id": "e679ed94-66a0-4d32-9823-e35373802e79",
                "createdAt": 1672909754.807,
                "message": "(service STAGING-SERVICE) has started 1 tasks: (task 5b834e71829e428abc0b6e0daaae97ca)."
            }
        ],
        "createdAt": 1672909747.903,
        "placementConstraints": [],
        "placementStrategy": [],
        "networkConfiguration": {
            "awsvpcConfiguration": {
                "subnets": [
                    "subnet-075a646c201c7b88c",
                    "subnet-07d38a366dfbba035",
                    "subnet-05e222a67edc6403a"
                ],
                "securityGroups": [
                    "sg-0a0d2d1fbfea4e1f1"
                ],
                "assignPublicIp": "ENABLED"
            }
        },
        "healthCheckGracePeriodSeconds": 0,
        "schedulingStrategy": "REPLICA",
        "deploymentController": {
            "type": "ECS"
        },
        "createdBy": "arn:aws:iam::866308211434:user/zhajili",
        "enableECSManagedTags": true,
        "propagateTags": "NONE"
    }
}

```

### Configure Production Cluster and Production Pipeline

In this pipeline ,we will only deploy our container to the ECS since it is production and all previous steps have been implemented in Staging ,hence in our Jenkinsfile, we will have only 1 stage which is deploying to ECS.

```
def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]
pipeline{
    agent any
    tools{
        maven "MAVEN3"
        jdk "JDK"
    }
    environment{

        ecs_cluster_name = "production"
        service_name = "PRODUCTION-SERVICE"

    }

    stages{
        
        stage("Deploy to ECS Production"){
            steps{
               withAWS(credentials: "cicdjenkins",region: "us-east-1" ){
                // force to delete old container deploy new one via service
                sh 'aws ecs update-service --cluster ${ecs_cluster_name} --service ${service_name} --force-new-deployment'
               }
            }

        }
    }
    post {
    always {
        echo 'Slack Notifications.'
        slackSend channel: '#jenkins',
            color: COLOR_MAP[currentBuild.currentResult],
            message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
            }
    }
}

```

[![Screenshot-2023-01-05-at-13-37-06.png](https://i.postimg.cc/X7sB5S36/Screenshot-2023-01-05-at-13-37-06.png)](https://postimg.cc/VdbNyhdK)




