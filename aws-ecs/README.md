# Prequisites

* A VPC with public and private subnets
* An ECS Cluster
* YScape Docker credentials in AWS Secrets Manager

```bash
# Create the secret
aws secretsmanager create-secret --name yscape-docker-credentials --description "YScape Docker credentials"
# 👆 Copy the secret ARN from the output

# Set the secret value to the docker credentials in the JSON format expected by AWS [1]
aws secretsmanager put-secret-value --secret-id arn:aws:secretsmanager:XXX --secret-string '{"username": "USERNAME", "password": "PASSWORD"}'
```

# Launch the YScape ECS Service

## 1. Configure Cloudformation Stack Parameters

Make a copy of the example file:

```bash
cp parameters.json.example parameters.json
```

And then update the parameters.json file with the following values:

* `VpcId`: The ID of the VPC in which the ECS Service will be launched
* `PrivateSubnetPrimary`: The ID of the private subnet in which the ECS Service will be launched
* `PrivateSubnetSecondary`: The ID of an optional, 2nd private subnet in which the ECS Service can be launched for High Availability
* `PublicSubnetIds`: The ID of the public subnets in which the ECS Service will be launched, separated by commas
* `ClusterName`: The name of the ECS Cluster
* `ServiceName`: The name of the ECS Service (Default: yscape-app)
* `DomainName`: The domain name pointing to the ECS Service (e.g. yscape.yourcompany.com)
* `YScapeDockerCredentialsSecretArn`: The ARN of the secret containing the docker credentials created in the Prerequisites step
* `YScapeDockerImage`: The URL of the YScape Docker image (e.g. registry.yscape.dev/yscape/app:latest)
* `ContainerCpu`: The CPU limit of the container (Default: 2048)
* `ContainerMemory`: The memory limit of the container (Default: 2048)

Store the parameters.json file somewhere so that it can be used in the future by the Cloudformation stack.

## 2. Create the Cloudformation Stack

Use the following command to create the Cloudformation stack:

```bash
aws cloudformation create-stack --capabilities CAPABILITY_IAM --stack-name yscape-app --template-body file://app.yml --parameters file://parameters.json
```

This will create the following resources:

* StorageVolume (AWS::EFS::FileSystem) - The EFS volume used to store the persisted state of the YScape app
    * MountTargetSecurityGroup (AWS::EC2::SecurityGroup)
    * MountTarget1 (AWS::EFS::MountTarget)
    * MountTarget2 (AWS::EFS::MountTarget) (conditional on the presence of a 2nd subnet specified in the `PrivateSubnetSecondary` parameter)
* Service (AWS::ECS::Service) - The ECS service used to run the YScape app
    * TaskDefinition (AWS::ECS::TaskDefinition)
    * ECSTaskExecutionRole (AWS::IAM::Role)
    * ServiceSecurityGroup (AWS::EC2::SecurityGroup)
    * LogGroup (AWS::Logs::LogGroup)
* PublicLoadBalancer (AWS::ElasticLoadBalancingV2::LoadBalancer) - The public load balancer used to expose the YScape app to the internet
    * ServiceTargetGroup (AWS::ElasticLoadBalancingV2::TargetGroup)
    * PublicLoadBalancerListener (AWS::ElasticLoadBalancingV2::Listener)
    * RedirectHTTPListener (AWS::ElasticLoadBalancingV2::Listener)
    * PublicLoadBalancerSecurityGroup (AWS::EC2::SecurityGroup)
* Certificate (AWS::CertificateManager::Certificate)

## 3. Verify the TLS Certificate

During the creation of the stack, the TLS certificate will need to be verified via the AWS Certificate Manager dashboard before stack creation can be completed [2].

## 4. Ensure the DNS Record for the Domain Name is Set

Ensure a CNAME record is set for the domain name pointing to the public load balancer's DNS name.

## 5. Access your YScape instance

Visit the domain name in your browser to access the YScape instance. Follow the onboarding process to complete the setup.

# Updating YScape

## When Using :latest or similar tag

Force a new deployment of the ECS service:

```bash
aws ecs update-service --cluster $(CLUSTER_NAME) --service $(SERVICE_NAME)
```

## When Pinning to a Specific Version

1. Update the parameters.json file with the new image in the `YScapeDockerImage` parameter
2. Update the Cloudformation stack

```bash
aws cloudformation update-stack --capabilities CAPABILITY_IAM --stack-name $(SERVICE_NAME) --template-body file://app.yml --parameters file://parameters.json
```

# References

[1] https://docs.aws.amazon.com/AmazonECS/latest/developerguide/private-auth.html
[2] https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-certificatemanager-certificate.html