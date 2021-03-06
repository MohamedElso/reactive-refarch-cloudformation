# Reactive Microservices Architectures with Amazon ECS, AWS Fargate, AWS Lambda, Amazon Kinesis Data Streams, Amazon ElastiCache, and Amazon DynamoDB

This reference architecture provides a set of YAML templates for deploying a reactive microservices architecture based on [Amazon Elastic Container Service (Amazon ECS)](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html) using [AWS Fargate](https://aws.amazon.com/fargate/), [AWS Lambda](https://aws.amazon.com/lambda/), [Amazon Kinesis Data Streams](https://aws.amazon.com/kinesis/streams/), [Amazon ElastiCache](https://aws.amazon.com/elasticache/), and [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) using [Amazon CloudFormation](https://aws.amazon.com/cloudformation/).

You can launch this CloudFormation stack in the US East (North Virginia) Region in your account:

[![cloudformation-launch-stack](images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=reactive&templateURL=https://s3.amazonaws.com/reactive-refarch-cloudformation-us-east-1/master.yaml)  

The Amazon CloudFormation-template is based on the [reference architecture](https://github.com/awslabs/ecs-refarch-cloudformation) for deploying containerized microservices with Amazon ECS (Fargate launch type) and AWS CloudFormation (YAML).

# Overview

![infrastructure-overview](images/architecture-overview.jpg)

The repository consists of a set of nested templates that deploy the following:

 - A tiered [VPC](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html) with public and private subnets, spanning an AWS region.
 - A highly available ECS cluster deployed across two [Availability Zones](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html) in Fargate mode.
 - A pair of [NAT gateways](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-nat-gateway.html) (one in each zone) to handle outbound traffic.
 - One microservice deployed as [ECS service](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html) implementing the base logic for an ad-tracking solution leveraging the non-blocking, async toolkit [Vert.x](http://vertx.io/).
 - An [Application Load Balancer (ALB)](https://aws.amazon.com/elasticloadbalancing/applicationloadbalancer/) to the public subnets to handle inbound traffic.
 - ALB path-based routing for the ECS service to route the inbound traffic to the correct service.
 - Centralized container logging with [Amazon CloudWatch Logs](http://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html).
 - Two [AWS Lambda](https://aws.amazon.com/lambda/)-functions to consume data from [Amazon Kinesis Streams](https://aws.amazon.com/kinesis/streams/) to process and store data.
 - Two [Amazon Kinesis Streams](https://aws.amazon.com/kinesis/streams/) to decouple microservies from each other.
 - A highly available [Amazon ElastiCache](https://aws.amazon.com/elasticache/)-cluster (Redis 5) deployed across two [Availability Zones](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html) in a [Replication Group with automatic failover](https://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/AutoFailover.html). Redis is used as central data store and pub/sub messaging implementation.
 - An [Amazon DynamoDB](https://aws.amazon.com/dynamodb/)-table to persist event-data.

## Why use AWS CloudFormation?

Using CloudFormation to deploy and manage services with ECS or Lambda has a number of nice benefits over more traditional methods ([AWS CLI](https://aws.amazon.com/cli), scripting, etc.).

### Infrastructure-as-Code

A template can be used repeatedly to create identical copies of the same stack (or to use as a foundation to start a new stack).  Templates are simple YAML- or JSON-formatted text files that can be placed under your normal source control mechanisms, stored in private or public locations such as Amazon S3, and exchanged via email. With CloudFormation, you can see exactly which AWS resources make up a stack. You retain full control and have the ability to modify any of the AWS resources created as part of a stack.

### Self-documenting

Fed up with outdated documentation on your infrastructure or environments? Still keep manual documentation of IP ranges, security group rules, etc.?

With CloudFormation, your template becomes your documentation. Want to see exactly what you have deployed? Just look at your template. If you keep it in source control, then you can also look back at exactly which changes were made and by whom.

### Intelligent updating & rollback

CloudFormation not only handles the initial deployment of your infrastructure and environments, but it can also manage the whole lifecycle, including future updates. During updates, you have fine-grained control and visibility over how changes are applied, using functionality such as [change sets](https://aws.amazon.com/blogs/aws/new-change-sets-for-aws-cloudformation/), [rolling update policies](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html) and [stack policies](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html).

## Template details

The templates below are included in this repository and reference architecture:

| Template | Description |
| --- | --- |
| [master.yaml](master.yaml) | This is the master template - deploy it to CloudFormation and it includes all of the others automatically. |
| [infrastructure/vpc.yaml](infrastructure/vpc.yaml) | This template deploys a VPC with a pair of public and private subnets spread across two Availability Zones. It deploys an [Internet gateway](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Internet_Gateway.html), with a default route on the public subnets. It deploys a pair of NAT gateways (one in each zone), and default routes for them in the private subnets. |
| [infrastructure/security-groups.yaml](infrastructure/security-groups.yaml) | This template contains the [security groups](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html) required by the entire stack. They are created in a separate nested template, so that they can be referenced by all of the other nested templates. |
| [infrastructure/load-balancers.yaml](infrastructure/load-balancers.yaml) | This template deploys an ALB to public subnets, which exposes the ECS service. It is created in in a separate nested template, so that it can be referenced by all of the other nested templates and so that the ECS service can register with it. |
| [infrastructure/ecs-cluster.yaml](infrastructure/ecs-cluster.yaml) | This template deploys an ECS cluster to the private subnets in Fargate mode. |
| [infrastructure/dynamodb.yaml](infrastructure/dynamodb.yaml) | This template creates a DynamoDB table to persist event-data. |
| [infrastructure/elasticache](infrastructure/elasticache.yaml) | This template deploys an ElastiCache cluster with Redis 3-engine to the private subnets with a Replication Group for high availability. |
| [infrastructure/kinesis](infrastructure/kinesis.yaml) | This template deploys two Kinesis Streams to decouple services from each other and implement a buffer to temporarily store data. |
| [infrastructure/lambda](infrastructure/lambda) | This template deploys two Lambda-functions with according EventSourceMapping to consume data from Kinesis Streams |
| [services/tracking-service/service.yaml](services/tracking-service/service.yaml) | This is an example of a long-running ECS service that implements the basic logic for an AdTracking use-case. For the full source for the service, see [services/tracking-service/src](services/tracking-service/src).|

After the CloudFormation templates have been deployed, the [stack outputs](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html) contain a link to the load-balanced URL for the deployed microservice.

![stack-outputs](images/stack-outputs.png)

## How do I...?

### Get started and deploy this into my AWS account

You can launch this CloudFormation stack in the US East 1 (North Virginia) Region in your account:

[![cloudformation-launch-button](images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=reactive&templateURL=https://reactive-refarch-cf-us-east-1.s3.amazonaws.com/master.yaml)    

### Test the application

After the application has been deployed correctly, you can load test data into Redis by calling the following URL using e.g. curl:

```
curl http://<endpoint>/cache/fill
```

After the cache has been filled succesfully, you can call the tracking application with an existing program id e.g. 212313

```
curl http://<endpoint>/event/212313
```

This HTTP call returns a response like

```
{"userAgent":"curl/7.54.0","programId":"212313","programName":"program2","checksum":"124","customerId":9124,"customerName":"Customer2","messageId":"06bc2944-886c-4e56-907c-fa248c8af023","valid":true"}
```

### Customize the templates

1. [Fork](https://github.com/awslabs/ecs-refarch-cloudformation#fork-destination-box) this GitHub repository.
2. Clone the forked GitHub repository to your local machine.
3. Modify the templates.
4. Upload them to an Amazon S3 bucket of your choice.
5. Either create a new CloudFormation stack by deploying the master.yaml template, or update your existing stack with your version of the templates.

### Build the microservices

All microservices a implemented using Java 8/11 and Golang and the Java microservices use Maven for dependency management.

#### Main application

The application has two different build-targets: standard build with Maven without any special paratemers and with the profile `native-image-fargate` which builds a native binary using [GraalVM](https://www.graalvm.org/). For this particular build target, it is necessary to also use a different Dockerfile (`Dockerfile-native`) which uses a [multistage build process](https://docs.docker.com/develop/develop-images/multistage-build/) to copy necessary files from the GraalVM-builder image to the target image (`libsunec.so` and `cacerts`), and to build the application inside the Docker container using GraalVM and Maven.

##### Standard build target

1. Change directory to `services/tracking-service/reactive-vertx`: `cd services/tracking-service/reactive-vertx`
2. Build an User-JAR using Maven: `mvn clean install -Dmaven.test.skip=true`
3. Build a Docker image: `docker build . -t <your_docker_repo>/reactive-vertx` -f Dockerfile

##### Native GraalVM build target

For this build target, it is not necessary to install GraalVM, because it uses a multi-stage Docker image to build the application inside of a Docker container.

1. Change directory to `services/tracking-service/reactive-vertx`: `cd services/tracking-service/reactive-vertx`
2. Build a Docker image: `docker build . -t <your_docker_repo>/reactive-vertx` -f Dockerfile-native

#### Redis Updater

1. Change directory to `services/redis-updater`: `cd services/redis-updater`
2. Build an ZIP file using make: `make build`

#### Kinesis Consumer

1. Change directory to `services/redis-updater`: `cd services/redis-updater`
2. Build an ZIP file using make: `make build`

### Create a new ECS service

1. Push your container to a registry somewhere (e.g., [Amazon ECR](https://aws.amazon.com/ecr/)).
2. Copy one of the existing service templates in [services/*](/services).
3. Update the `ContainerName` and `Image` parameters to point to your container image instead of the example container.
4. Increment the `ListenerRule` priority number (no two services can have the same priority number - this is used to order the ALB path based routing rules).
5. Copy one of the existing service definitions in [master.yaml](master.yaml) and point it at your new service template. Specify the HTTP `Path` at which you want the service exposed. 
6. Deploy the templates as a new stack, or as an update to an existing stack.

### Setup centralized container logging

By default, the containers in your ECS tasks/services are already configured to send log information to CloudWatch Logs and retain them for 7 days. Within each service's template (in [services/*](services/)), a LogGroup is created that is named after the CloudFormation stack. All container logs are sent to that CloudWatch Logs log group.

You can view the logs by looking in your [CloudWatch Logs console](https://console.aws.amazon.com/cloudwatch/home?#logs:) (make sure you are in the correct AWS region).

ECS also supports other logging drivers, including `syslog`, `journald`, `splunk`, `gelf`, `json-file`, and `fluentd`. To configure those instead, adjust the service template to use the alternative `LogDriver`. You can also adjust the log retention period from the default 365 days by tweaking the `RetentionInDays` parameter.

For more information, see the [LogConfiguration](http://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_LogConfiguration.html) API operation.

### Deploy multiple environments (e.g., dev, test, pre-production)

Deploy another CloudFormation stack from the same set of templates to create a new environment. The stack name provided when deploying the stack is prefixed to all taggable resources (e.g., EC2 instances, VPCs, etc.) so you can distinguish the different environment resources in the AWS Management Console. 

### Change the VPC or subnet IP ranges

This set of templates deploys the following network design:

| Item | CIDR Range | Usable IPs | Description |
| --- | --- | --- | --- |
| VPC | 10.180.0.0/16 | 65,536 | The whole range used for the VPC and all subnets |
| Public Subnet | 10.180.8.0/21 | 2,041 | The public subnet in the first Availability Zone |
| Public Subnet | 10.180.16.0/21 | 2,041 | The public subnet in the second Availability Zone |
| Private Subnet | 10.180.24.0/21 | 2,041 | The private subnet in the first Availability Zone |
| Private Subnet | 10.180.32.0/21 | 2,041 | The private subnet in the second Availability Zone |

You can adjust the CIDR ranges used in this section of the [master.yaml](master.yaml) template:

```
VPC:
  Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub ${TemplateLocation}/infrastructure/vpc.yaml
      Parameters:
        EnvironmentName:    !Ref AWS::StackName
        VpcCIDR:            10.180.0.0/16
        PublicSubnet1CIDR:  10.180.8.0/21
        PublicSubnet2CIDR:  10.180.16.0/21
        PrivateSubnet1CIDR: 10.180.24.0/21
        PrivateSubnet2CIDR: 10.180.32.0/21
```

### Update an ECS service to a new Docker image version

ECS has the ability to perform rolling upgrades to your ECS services to minimize downtime during deployments. For more information, see [Updating a Service](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/update-service.html).

To update one of your services to a new version, adjust the `Image` parameter in the service template (in [services/*](services/) to point to the new version of your container image. For example, if `1.0.0` was currently deployed and you wanted to update to `1.1.0`, you could update it as follows:

```
TaskDefinition:
  Type: AWS::ECS::TaskDefinition
  Properties:
    ContainerDefinitions:
      - Name: your-container
        Image: registry.example.com/your-container:1.1.0
```

After you've updated the template, update the deployed CloudFormation stack; CloudFormation and ECS handle the rest. 

To adjust the rollout parameters (min/max number of tasks/containers to keep in service at any time), you need to configure `DeploymentConfiguration` for the ECS service.

For example:

```
Service:
  Type: AWS::ECS::Service
    Properties:
      ...
      DesiredCount: 4
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 50
```

### Add a new item to this list

If you found yourself wishing this set of frequently asked questions had an answer for a particular problem, please [submit a pull request](https://help.github.com/articles/creating-a-pull-request-from-a-fork/). The chances are that others will also benefit from having the answer listed here.

## Contributing

Please [create a new GitHub issue](https://github.com/awslabs/ecs-refarch-cloudformation/issues/new) for any feature requests, bugs, or documentation improvements. 

Where possible, please also [submit a pull request](https://help.github.com/articles/creating-a-pull-request-from-a-fork/) for the change.

