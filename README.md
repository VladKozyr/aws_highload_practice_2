# KMA AWS practice

## Create VPC
- Create
```
aws ec2 create-vpc --cidr-block 10.10.0.0/18 --no-amazon-provided-ipv6-cidr-block --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=kma-genesis},{Key=Lesson,Value=public-clouds}]' --query Vpc.VpcId --output text
```
- Get VPC ID
```
aws ec2 describe-vpcs | grep "VpcId"
```

## Create subnets
- Create subnets with ZONE and VPC_ID, in my case us-east-2 (Ohio)
```
aws ec2 create-subnet --vpc-id $VPC_ID --availability-zone $ZONE --cidr-block 10.10.1.0/24
aws ec2 create-subnet --vpc-id $VPC_ID --availability-zone $ZONE --cidr-block 10.10.2.0/24
aws ec2 create-subnet --vpc-id $VPC_ID --availability-zone $ZONE --cidr-block 10.10.3.0/24
```

## Internet gateway
- Create internet gateway
```
aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text          
```
- Attach gateway to vpc by IGW_ID and VPC_ID
```
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID 
```

## Create security groups for ports
- Create security group
```
aws ec2 create-security-group --group-name kma-security-group --description "KMA SG" --vpc-id $VPC_ID
```
- Attach ports to security group
```
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr  0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr  0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 443 --cidr  0.0.0.0/0
```

## Add load balancer
- Create ec2 launch template
```
aws ec2 create-launch-template --launch-template-name KMA_HIGHLOAD_TEMPLATE --version-description TEMPLATE_VER_1.0 --region us-east-2 --launch-template-data '$DATA'
```
Where $DATA:
```json
{
   "NetworkInterfaces":[
      {
         "DeviceIndex":0,
         "AssociatePublicIpAddress":true,
         "Groups":[
            "sg-00f7ee321c5bc9fd2"
         ],
         "DeleteOnTermination":true
      }
   ],
   "ImageId":"$AMI_LINUX_ID",
   "InstanceType":"t3.micro",
   "TagSpecifications":[
      {
         "ResourceType":"instance",
         "Tags":[
            {
               "Key":"Name",
               "Value":"KMA_LAUNCH_TEMPLATE"
            }
         ]
      }
   ],
   "BlockDeviceMappings":[
      {
         "DeviceName":"/dev/sda1",
         "Ebs":{
            "VolumeSize":15
         }
      }
   ]
}
```
AMI_LINUX_ID for specific zone, in my case - **ami-0b59bfac6be064b78**
- Create autoscaling group
```
aws autoscaling create-auto-scaling-group --auto-scaling-group-name KMA_ASG --launch-template "LaunchTemplateName=KMA_HIGHLOAD_TEMPLATE" --min-size 1 --max-size 2 --desired-capacity 1 --vpc-zone-identifier "$SUB_1, $SUB_2, $SUB_3" --availability-zones "us-east-2a" "us-east-2b" "us-east-2c"
```
- Create load balancer
```
aws elbv2 create-load-balancer --name KMA_LOAD_BALANCER  --subnets $SUB_1 $SUB_2 $SUB_3 --security-groups $SG_ID
```
