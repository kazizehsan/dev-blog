---
title: ECS Fargate Task as a Web server
date: 2023-12-03 22:42:00 +0600
categories: [Guides]
tags: [deployment, aws, ecr, load-balancer, security-group, ecs, fargate, task, cloudformation]
---

![Desktop View](/assets/img/bg/c-dustin-K-Iog-Bqf8E-unsplash.jpg){: w="700" h="400" }
_Photo by <a href="https://unsplash.com/@dianamia?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">C Dustin</a> on <a href="https://unsplash.com/photos/white-clouds-K-Iog-Bqf8E?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>_


You can manually run a one-off ECS Fargate Task based off a Dockerized image of your API server. But if you are expecting heavy traffic, you can setup Load Balancer, Service etc. In this guide we will setup everything using CloudFormation.

Some key points to note are:
* we will create a Load Balancer with its own Security Group
* we will create a Service with its own Security Group (Tasks start automatically on creation of Service)
* we will modify Security Group (inbound rule) of Service to receive traffic from Security Group of Load Balancer

> When Task Definition is revised (essentially registering the task def again) with a new ECR image tag, the old Task running in the Service is stopped and its Public IP is stripped away. After a while the Task is removed. This Public IP was actually never directly consumed since the user accesses by the Load Balancer's IP.
{: .prompt-info }
> Existing tasks and services that reference a DELETE_IN_PROGRESS task definition revision continue to run without disruption.
{: .prompt-info }
> If you stop a Task managed by a Service, a new instance of the Task will start again automatically if the Service has a target of running at least 1 Task at all times.
{: .prompt-info }

## Prerequisites

In AWS Management Console, use the following settings to create a Security Group for the Load Balancer.

* Name: APILoadBalancerSG
* Inbound rule: 
    * Type: Custom TCP
    * Protocol: TCP
    * Port range: 80
    * Source: 0.0.0.0/0

Another SG for the ECS Service.

* Name: APIServiceSG
* Inbound rule: 
    * Type: Custom TCP
    * Protocol: TCP
    * Port range: 80
    * Source: APILoadBalancerSG


> These two SGs needed to be created beforehand because the API server in this example expected an RDS to preexist since alembic migrations are done on API task startup and the RDS instance needed a SG that referenced the APIServiceSG.
{: .prompt-info }

## CloudFormation

```yaml
AWSTemplateFormatVersion: 2010-09-09
Parameters:
  DeploymentEnv:
    Description: The environment for deployment
    Type: String
    AllowedValues:
      - prod
  VPC:
    Type: AWS::EC2::VPC::Id
  SubnetA:
    Type: AWS::EC2::Subnet::Id
  SubnetB:
    Type: AWS::EC2::Subnet::Id
  ContainerSecurityGroup: # the APIServiceSG created earlier
    Type: AWS::EC2::SecurityGroup::Id
  LoadBalancerSecurityGroup: # the APILoadBalancerSG created earlier
    Type: AWS::EC2::SecurityGroup::Id
Resources:
  Cluster:
    Type: 'AWS::ECS::Cluster'
    Properties:
      ClusterName: MyAPICluster
      CapacityProviders:
        - FARGATE
  TaskExecutionRole:
    Type: 'AWS::IAM::Role'
    Properties:
      RoleName: MyAPITaskExecutionRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ecs-tasks.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Policies:
        - PolicyName: MyAPITaskExecutionRolePolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: 
                  - 'ecr:GetAuthorizationToken'
                  - 'ecr:BatchCheckLayerAvailability'
                  - 'ecr:GetDownloadUrlForLayer'
                  - 'ecr:BatchGetImage'
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                  - 'ssm:DescribeParameters'
                  - 'ssm:GetParameterHistory'
                  - 'ssm:GetParametersByPath'
                  - 'ssm:GetParameters'
                  - 'ssm:GetParameter'
                Resource: '*'
  TaskRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: MyAPITaskRole
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: 'sts:AssumeRole'
      Policies:
        - PolicyName: MyAPITaskRolePolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: AllowSSM
                Effect: Allow
                Action:
                  - 'ssm:DescribeParameters'
                  - 'ssm:GetParameterHistory'
                  - 'ssm:GetParametersByPath'
                  - 'ssm:GetParameters'
                  - 'ssm:GetParameter'
                  - 'ssm:PutParameter'
                Resource: '*'
              - Sid: AllowLogs
                Effect: Allow
                Action:
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                Resource: '*'
              - Sid: AllowECR
                Effect: Allow
                Action:
                  - 'ecr:GetAuthorizationToken'
                  - 'ecr:BatchCheckLayerAvailability'
                  - 'ecr:GetDownloadUrlForLayer'
                  - 'ecr:BatchGetImage'
                  - 'ecr:DescribeImages'
                  - 'ecr:GetRepositoryPolicy'
                  - 'ecr:SetRepositoryPolicy'
                Resource:
                  - !Sub 'arn:aws:ecr:${AWS::Region}:${AWS::AccountId}:repository/*'
  LogGroup: 
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /ecs/MyAPITaskDef
      RetentionInDays: 30
  TaskDefinition:
    Type: 'AWS::ECS::TaskDefinition'
    DependsOn:
      - LogGroup
      - TaskExecutionRole
      - TaskRole
    Properties:
      Family: MyAPITaskDef
      ContainerDefinitions:
        - EntryPoint:
            - sh
            - '-c'
          Command:
            - >-
              /bin/bash -c "
              alembic upgrade head;
              gunicorn src.main:app --workers 1 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:80 --log-level=debug --timeout=90"
          Essential: true
          Image: !Join
            - ''
            - - !Ref AWS::AccountId
              - .dkr.ecr.
              - !Ref AWS::Region
              - .amazonaws.com/
              - my-api-image:latest
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: ecs
          Name: MyAPIContainer
          PortMappings:
            - ContainerPort: 80
              Protocol: tcp
          Environment:
            - Name: ENV
              Value: !Ref DeploymentEnv
          Secrets:
            - Name: POSTGRES_HOST
              ValueFrom: !Sub '/myapp/${DeploymentEnv}/db_host'
            - Name: POSTGRES_DB
              ValueFrom: !Sub '/myapp/${DeploymentEnv}/db_name'
            - Name: POSTGRES_PASSWORD
              ValueFrom: !Sub '/myapp/${DeploymentEnv}/db_password'
            - Name: POSTGRES_PORT
              ValueFrom: !Sub '/myapp/${DeploymentEnv}/db_port'
            - Name: POSTGRES_USER
              ValueFrom: !Sub '/myapp/${DeploymentEnv}/db_user'
            - Name: AUTH0_DOMAIN
              ValueFrom: !Sub '/myapp/${DeploymentEnv}/auth0/domain'
      Cpu: 512
      ExecutionRoleArn: !Ref TaskExecutionRole
      TaskRoleArn: !Ref TaskRole
      Memory: 1024
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
  LoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value: 60
      Name: MyAPILoadBalancer
      Scheme: internet-facing
      SecurityGroups:
        - !Ref LoadBalancerSecurityGroup
      Subnets:
        - !Ref SubnetA
        - !Ref SubnetB
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckIntervalSeconds: 30
      HealthCheckPath: /v1/ping
      HealthCheckTimeoutSeconds: 6
      UnhealthyThresholdCount: 2
      HealthyThresholdCount: 2
      Name: MyAPITargetGroup
      Port: 80
      Protocol: HTTP
      TargetGroupAttributes:
        - Key: deregistration_delay.timeout_seconds
          Value: 60
      TargetType: ip
      VpcId: !Ref VPC
  ListenerHTTP:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - TargetGroupArn: !Ref TargetGroup
          Type: forward
      LoadBalancerArn: !Ref LoadBalancer
      Port: 80
      Protocol: HTTP
  Service:
    Type: 'AWS::ECS::Service'
    DependsOn:
      - ListenerHTTP
    Properties:
      ServiceName: MyAPIService
      Cluster: !Ref Cluster
      DesiredCount: 1
      HealthCheckGracePeriodSeconds: 60
      LaunchType: FARGATE
      LoadBalancers:
        - ContainerName: MyAPIContainer
          ContainerPort: 80
          TargetGroupArn: !Ref TargetGroup
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: ENABLED
          Subnets:
            - !Ref SubnetA
            - !Ref SubnetB
          SecurityGroups:
            - !Ref ContainerSecurityGroup
      TaskDefinition: !Ref TaskDefinition
Outputs:
  Endpoint:
    Description: Endpoint
    Value: !Join 
      - ''
      - - http://
        - !GetAtt LoadBalancer.DNSName
```
