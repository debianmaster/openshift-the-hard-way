```console
aws configure  #ca-central-1  
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.240.0.0/16 | \
  jq -r '.Vpc.VpcId')
aws ec2 create-tags \
  --resources ${VPC_ID} \
  --tags Key=Name,Value=kubernetes
aws ec2 modify-vpc-attribute \
  --vpc-id ${VPC_ID} \
  --enable-dns-support '{"Value": true}'
aws ec2 modify-vpc-attribute \
  --vpc-id ${VPC_ID} \
  --enable-dns-hostnames '{"Value": true}'

DHCP_OPTION_SET_ID=$(aws ec2 create-dhcp-options \
  --dhcp-configuration "Key=domain-name,Values=us-west-2.compute.internal" \
    "Key=domain-name-servers,Values=AmazonProvidedDNS" | \
  jq -r '.DhcpOptions.DhcpOptionsId')
aws ec2 create-tags \
  --resources ${DHCP_OPTION_SET_ID} \
  --tags Key=Name,Value=kubernetes
aws ec2 associate-dhcp-options \
  --dhcp-options-id ${DHCP_OPTION_SET_ID} \
  --vpc-id ${VPC_ID}

SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.240.0.0/24 | \
  jq -r '.Subnet.SubnetId')
aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=kubernetes



INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway | \
  jq -r '.InternetGateway.InternetGatewayId')
aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=kubernetes
aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}


SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name kubernetes \
  --description "Kubernetes security group" \
  --vpc-id ${VPC_ID} | \
  jq -r '.GroupId')
aws ec2 create-tags \
  --resources ${SECURITY_GROUP_ID} \
  --tags Key=Name,Value=kubernetes
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol all
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol all \
  --port 0-65535 \
  --cidr 10.240.0.0/16
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

aws elb create-load-balancer \
  --load-balancer-name kubernetes \
  --listeners "Protocol=TCP,LoadBalancerPort=443,InstanceProtocol=TCP,InstancePort=443" \
  --subnets ${SUBNET_ID} \
  --security-groups ${SECURITY_GROUP_ID}


cat > kubernetes-iam-role.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com"}, "Action": "sts:AssumeRole"}
  ]
}
EOF
aws iam create-role \
  --role-name kubernetes \
  --assume-role-policy-document file://kubernetes-iam-role.json
cat > kubernetes-iam-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["ec2:*"], "Resource": ["*"]},
    {"Effect": "Allow", "Action": ["elasticloadbalancing:*"], "Resource": ["*"]},
    {"Effect": "Allow", "Action": ["route53:*"], "Resource": ["*"]},
    {"Effect": "Allow", "Action": ["ecr:*"], "Resource": "*"}
  ]
}
EOF
aws iam put-role-policy \
  --role-name kubernetes \
  --policy-name kubernetes \
  --policy-document file://kubernetes-iam-policy.json
aws iam create-instance-profile \
  --instance-profile-name kubernetes 
aws iam add-role-to-instance-profile \
  --instance-profile-name kubernetes \
  --role-name kubernetes



IMAGE_ID="ami-daeb57be"


aws ec2 create-key-pair --key-name kubernetes | \
  jq -r '.KeyMaterial' > ~/.ssh/kubernetes_the_hard_way
chmod 600 ~/.ssh/kubernetes_the_hard_way
ssh-add ~/.ssh/kubernetes_the_hard_way 


CONTROLLER_0_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.medium \
  --private-ip-address 10.240.0.10 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')

aws ec2 modify-instance-attribute \
  --instance-id ${CONTROLLER_0_INSTANCE_ID} \
  --no-source-dest-check
aws ec2 create-tags \
  --resources ${CONTROLLER_0_INSTANCE_ID} \
  --tags Key=Name,Value=controller0

CONTROLLER_1_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.medium \
  --private-ip-address 10.240.0.11 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
aws ec2 modify-instance-attribute \
  --instance-id ${CONTROLLER_1_INSTANCE_ID} \
  --no-source-dest-check
aws ec2 create-tags \
  --resources ${CONTROLLER_1_INSTANCE_ID} \
  --tags Key=Name,Value=controller1

CONTROLLER_2_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.medium \
  --private-ip-address 10.240.0.12 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
  
aws ec2 modify-instance-attribute \
  --instance-id ${CONTROLLER_2_INSTANCE_ID} \
  --no-source-dest-check
aws ec2 create-tags \
  --resources ${CONTROLLER_2_INSTANCE_ID} \
  --tags Key=Name,Value=controller2

WORKER_0_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.medium \
  --private-ip-address 10.240.0.20 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
aws ec2 modify-instance-attribute \
  --instance-id ${WORKER_0_INSTANCE_ID} \
  --no-source-dest-check
aws ec2 create-tags \
  --resources ${WORKER_0_INSTANCE_ID} \
  --tags Key=Name,Value=worker0
WORKER_1_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.medium \
  --private-ip-address 10.240.0.21 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
aws ec2 modify-instance-attribute \
  --instance-id ${WORKER_1_INSTANCE_ID} \
  --no-source-dest-check
aws ec2 create-tags \
  --resources ${WORKER_1_INSTANCE_ID} \
  --tags Key=Name,Value=worker1
WORKER_2_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.medium \
  --private-ip-address 10.240.0.22 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
aws ec2 modify-instance-attribute \
  --instance-id ${WORKER_2_INSTANCE_ID} \
  --no-source-dest-check
aws ec2 create-tags \
  --resources ${WORKER_2_INSTANCE_ID} \
  --tags Key=Name,Value=worker2
  
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" | \
  jq -j '.Reservations[].Instances[] | .InstanceId, "  ", .Placement.AvailabilityZone, "  ", .PrivateIpAddress, "  ", .PublicIpAddress, "\n"'
  

```


