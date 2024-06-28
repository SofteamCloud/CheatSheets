# aws-cli tips

## AWS EC2 

### Get id of running instance
```sh
aws ec2 describe-instances \
                        --filters "Name=tag:Name,Values=ec2-$ENV*" "Name=instance-state-name,Values=running"\
                        --output text \
                        --query 'Reservations[0].Instances[0].InstanceId'
```

### Get EC2 windows password   
```sh
aws ec2 get-password-data --instance-id $instance --query PasswordData --output text \
            | base64 -D | openssl rsautl -decrypt -inkey ~/.ssh/key
```

### Get EC2 public IP 

```sh

aws ec2 describe-instances  --filters "Name=tag:Name,Values=<<search>>" \
                                      "Name=instance-state-name,Values=running" \
                                    --output text \
                                    --query 'Reservations[0].Instances[0].PublicIpAddress'
```

### (re)Start up instance by name
Example to boot my cloud9 instance from the cli, I use this alias: 
```sh
alias devUp='aws ec2 start-instances --instance-id=$(aws ec2 describe-instances  --filters "Name=tag:Name,Values=aws-cloud9-remoteDev"   --output text -query "Reservations[0].Instances[0].InstanceId")'
```

### Get multi value from query 
```sh
aws ec2 describe-network-interfaces --query 'NetworkInterfaces[*], {val1:NetworkInterfaceId, val2:Attachment.AttachmentId}' --output json
```


### port forward for rdp 
```sh 
aws ssm start-session --target <instance-id> --document-name AWS-StartPortForwardingSession --parameters "localPortNumber=55678,portNumber=3389"
```

## AWS S3 
### S3 create bucket 
```sh
export S3_BUCKET_NAME=transcribefr
aws s3api create-bucket --bucket $S3_BUCKET_NAME --region eu-west-3 --create-bucket-configuration LocationConstraint=eu-west-3
```

### sync it locally using : 
```sh
aws s3 sync s3://$S3_BUCKET_NAME debugBucket
```

###  S3 push recursif
`aws s3 cp $(ls -d ./results/ehs*) s3://s3-gatling-results-bucket/$REPORT_NAME --recursive`

### S3 presign 
```
aws s3 presign s3://bucket-file/file.txt --region eu-west-1
```

### empty delete all bucket with connect in name
```sh
aws s3 ls | awk '{print $3}' | grep connect | while read x; do  echo "deleting $x" ; aws s3 rm s3://$x --recursive && aws s3 rb s3://$x --force; done
```

### pre-signed URL
To create a pre-signed URL with a custom lifetime that links to an object in an S3 bucket

pre-signed URL for a specified bucket and key that is valid for one week:
```sh
aws s3 presign s3://awsexamplebucket/test2.txt --expires-in 604800
```


## AWS SSM 

### SSM send command
```sh
aws ssm send-command \
      --instance-ids $(aws ec2 describe-instances \
                        --filters "Name=tag:Name,Values=ec2-$ENV*" "Name=instance-state-name,Values=running"\
                        --output text \
                        --query 'Reservations[0].Instances[0].InstanceId') \
      --document-name "AWS-RunShellScript" \
      --comment "apply conf on $ENV" \
      --parameters '{"commands":["./update_efs.sh '$arg1' '$arg2' "],"executionTimeout":["600"],"workingDirectory":["/home/ec2-user"]}'
```

### How to get list of Region or services  

#### Regions 
```sh
 aws ssm get-parameters-by-path --path /aws/service/global-infrastructure/regions --output json |jq ".Parameters[].Name" | sort
```


#### List of service 

##### All services and description
```sh
curl "https://aws.amazon.com/api/dirs/typeahead-suggestions/items?locale=en_US&limit=500" | jq  '[.items[].additionalFields | select (.category == "products")]'
```

#####  List of service per region
```sh
aws ssm get-parameters-by-path --path /aws/service/global-infrastructure/regions/eu-west-3/services --output json | jq ".Parameters[].Name" | sort
```
######  List of regions where a service (Amazon Athena, in this case) is available:
```sh
aws ssm get-parameters-by-path --path /aws/service/global-infrastructure/services/athena/regions --output json | jq ".Parameters[].Value"
```

#### Get free tier usage 
```sh
 aws freetier get-free-tier-usage --profile mgt
```

#### Get ami id 
```sh
aws ssm get-parameter --region us-west-2 --name "/aws/service/bottlerocket/aws-k8s-1.15/x86_64/latest/image_id" --query Parameter.Value --output text
```

## AWS lambda 

### Get lambdas runtimes
```sh
aws lambda list-functions --function-version ALL --output text --query "Functions[?Runtime=='nodejs10.x'].FunctionArn"
```

## CloudFormation

### Searching stack name by partial name
in this exemple searching `amplif-<envVar>*` stack name 

```sh
aws cloudformation describe-stacks --output text --query 'Stacks[?contains(StackName, `amplify-'"$STACK_NAME"'`)].StackName'
```

### Wait for stack to be delete
```sh
aws cloudformation delete-stack --stack-name $STACK_NAME
aws cloudformation wait stack-delete-complete --stack-name $STACK_NAME
```

### bulk delete stack by search 
```sh
 
aws cloudformation describe-stacks --output table --query 'Stacks[?starts_with(StackName, `'"$MY_SEARCH"'`)].StackName' | awk 'NR>2 {print$2}' | grep -v ^$ |while read x; do  echo "deleting $x":; aws cloudformation delete-stack --stack-name $x ; done
```

### Enhance your cfn usage with rain
A simple wrapper for cfn commands
[see more here](https://github.com/aws-cloudformation/rain)
output sample 
```sh
rain deploy CMKExemple.yaml testCMK --region us-east-1
Stack testCMK:
  + AWS::CloudWatch::Alarm DisabledOrDeletedCmksAlarm
  + AWS::KMS::Alias KeyAlias
  + AWS::KMS::Key Key
  + AWS::Logs::MetricFilter MetricFilter
Do you wish to continue? (Y/n) Y
Deploying template 'CMKExemple.yaml' as stack 'testCMK' in us-east-1.
Stack testCMK: CREATE_COMPLETE
  Outputs:
    KeyId: 0ebbd05c-30a6-4aa0-a92b-XXXXX # Key id. (exported as testCMK-CMKKey-KeyId)
```

## ACM

### Create certificate 
you can create certification by another stack or by cli 
-  **via cli** `aws acm request-certificate --domain-name <<my-own-domain>> --validation-method DNS --region us-east-1` and validate via [console](https://console.aws.amazon.com/acm/home?region=us-east-1)   
-  **via cfn (auto validation)** 
```yaml
Resources:
  acmCertificate: 
    Type: "AWS::CertificateManager::Certificate"
    Properties: 
      DomainName: !Ref targetdnsName
      DomainValidationOptions:
            - DomainName: !Ref targetdnsName
              HostedZoneId: !Ref hostedZoneIdMainDomain
      ValidationMethod: DNS 

Outputs:
  acmCertificate:
    Description: 'DNS url'
    Value: !Ref acmCertificate
```

## Bedrock 

# Find exact model name for IaC 

For a More Specific Search for Claude Models
```sh
aws bedrock list-foundation-models --query 'modelSummaries[?providerName==`Anthropic`]'
```

## Cleaning

### Delete ecr by name
```sh
 aws ecr describe-repositories | jq -r '.repositories[].repositoryName' | egrep '\/(vimubuntu|app)$' | xargs -L1 aws ecr delete-repository --force --repository-name
```

###  LogGroups sorted by size
```sh
aws logs describe-log-groups --query "logGroups[*].{LogGroup:logGroupName,VolumeSize:storedBytes} | reverse(sort_by(@, &VolumeSize))" --output table
```

###  LogGroups sorted by rentention 
```sh
aws logs describe-log-groups --query "logGroups[*].{LogGroup:logGroupName,VolumeSize:storedBytes,RetentionInDays:retentionInDays} | reverse(sort_by(@, &VolumeSize))" --output table
```


### Delete awsloggroup
*WARN: set your default region first*
#### Delete all awsloggroup
```sh
aws logs describe-log-groups --query 'logGroups[*].logGroupName' --output table | \
awk '{print $2}' | grep -v ^$ | while read x; do  echo "deleting $x" ; aws logs delete-log-group --log-group-name $x; done
```
#### Only delete loggroup name starting with /aws/lambda
```sh
aws logs describe-log-groups --query 'logGroups[*].logGroupName' --output table | \
awk '{print $2}' | grep ^/aws/lambda | while read x; do  echo "deleting $x" ; aws logs delete-log-group --log-group-name $x; done
```

### Empty and delete all S3 bucket by search

#### Without versioning 
```sh
aws s3 ls | awk '{print $3}' | grep $yourSearch | while read x; do  echo "deleting $x" ; aws s3 rm s3://$x --recursive && aws s3 rb s3://$x --force; done
```

#### Delete all files with versioning enable 
*console display size is important zoom out*  
```sh
export yourSearch=<bucket name>
$(aws --output text s3api list-object-versions --bucket $yourSearch | grep -E "^VERSIONS" |awk -v b=$yourSearch '{print "aws s3api delete-object --key "$4" --version-id "$8" --bucket "b " ; "}') \
$(aws --output text s3api list-object-versions --bucket $yourSearch | grep -E "^DELETEMARKERS" |awk -v b=$yourSearch '{print "aws s3api delete-object --key "$3" --version-id "$5" --bucket "b " ;"}')

```

### Delete stacks by name 
```sh 
aws cloudformation describe-stacks --output table --query 'Stacks[?starts_with(StackName, `'"$MY_SEARCH"'`)].StackName' | awk 'NR>2 {print$2}' | grep -v ^$ |while read x; do  echo "deleting $x":; aws cloudformation delete-stack --stack-name $x ; done
```

## Export existing master/member hierarchy from GuardDuty.  

```sh
aws guardduty list-members --detector-id <Detector ID> --query "Members[].[AccountId, Email]" --output text | awk '{print $1 "," $2}' 
```
The output of the above command will be your GuardDuty member account IDs and their corresponding root email addresses.

## Get AWS account info from local
```sh
export AWS_ACCESS_KEY_ID="$(aws configure get aws_access_key_id --profile default)"
export AWS_SECRET_ACCESS_KEY="$(aws configure get aws_secret_access_key --profile default)"
export AWS_DEFAULT_REGION="$(aws configure get region --profile default)"
export AWS_SESSION_TOKEN="$(aws configure get aws_session_token)"
```