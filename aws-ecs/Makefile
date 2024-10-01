MAKEFLAGS += --always-make

CLUSTER_NAME=$(shell jq -r '.[] | select(.ParameterKey == "ClusterName") | .ParameterValue' parameters.json)
SERVICE_NAME=$(shell jq -r '.[] | select(.ParameterKey == "ServiceName") | .ParameterValue' parameters.json)

### Update Stack Commands
update-yscape-app:
	aws cloudformation update-stack \
		--stack-name $(SERVICE_NAME) \
		--capabilities CAPABILITY_IAM \
		--template-body file://app.yml \
		--parameters file://parameters.json

deploy-yscape-app:
	aws ecs update-service --cluster $(CLUSTER_NAME) --service $(SERVICE_NAME) --force-new-deployment

update-yscape-tasks:
	aws cloudformation update-stack --capabilities CAPABILITY_IAM --stack-name yscape-tasks --template-body file://tasks.yml


### Operational Commands
# Set Docker Credentials, get secret-name from output of app.yml
set-yscape-docker-credentials:
		aws secretsmanager update-secret --secret-id yscape-docker-credentials --secret-string '{"username": "$(USERNAME)", "password": "$(PASSWORD)"}'

start-task-for-console:
	aws ecs run-task --cluster $(CLUSTER_NAME) \
		--task-definition $(SERVICE_NAME) \
		--enable-execute-command \
		--network-configuration '{"awsvpcConfiguration": {"subnets":["subnet-0234cd97af2217288"],"securityGroups":["sg-0711b022d7b59b384"]}}' \
		--overrides '{ "containerOverrides": [{ "name": "yscape-app", "command": ["sleep", "3600"], "cpu": 768 }], "cpu": "768"}'

stop-task-for-console:
	aws ecs stop-task --cluster $(CLUSTER_NAME) --task $(TASK)

check-ecs-exec:
	~/Code/amazon-ecs-exec-checker/check-ecs-exec.sh $(CLUSTER_NAME) $(TASK)

console:
	aws ecs execute-command --cluster $(CLUSTER_NAME) --task $(TASK) --command "/bin/bash" --interactive

DEBUGGING_INSTANCE=i-0f3f909305233f95b
DEBUGING_INSTANCE_PUBLIC_DNS_NAME=ec2-34-220-111-229.us-west-2.compute.amazonaws.com
EFS_MOUNT_TARGET_IP=10.192.20.111

ec2-connect:
	aws ec2-instance-connect send-ssh-public-key \
		--availability-zone us-west-2a \
		--instance-id $(DEBUGGING_INSTANCE) \
		--instance-os-user ec2-user \
		--ssh-public-key file:///Users/jason/.ssh/id_ed25519.pub
	ssh -o "IdentitiesOnly=yes" -i ~/.ssh/id_ed25519 ec2-user@$(DEBUGING_INSTANCE_PUBLIC_DNS_NAME)

ec2-connect-nfs-tunnel:
	aws ec2-instance-connect send-ssh-public-key \
		--availability-zone us-west-2a \
		--instance-id $(DEBUGGING_INSTANCE) \
		--instance-os-user ec2-user \
		--ssh-public-key file:///Users/jason/.ssh/id_ed25519.pub
	ssh -N -L 2049:$(EFS_MOUNT_TARGET_IP):2049 \
		-o "IdentitiesOnly=yes" -i ~/.ssh/id_ed25519 ec2-user@$(DEBUGING_INSTANCE_PUBLIC_DNS_NAME)
#  MacOS: sudo mount -t nfs -o vers=4,tcp,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 -w 127.0.0.1:/ tmp/mnt/yscape_files
#  Linux: sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 10.192.20.154:/ /mnt/yscape_files


### Create Stack Commands ###
create-yscape-app:
	aws cloudformation create-stack --capabilities CAPABILITY_IAM --stack-name $(SERVICE_NAME) --template-body file://app.yml --parameters file://parameters.json

create-debugging-instance:
	aws cloudformation create-stack --stack-name $(SERVICE_NAME)-debugging-instance --template-body file://debug.yml --parameters file://parameters-debug.json

create-yscape-tasks:
	aws cloudformation create-stack --capabilities CAPABILITY_IAM --stack-name yscape-tasks --template-body file://tasks.yml 

