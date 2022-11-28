
#  Create new AWS user groups, users and access policies 

Step 1: [Create root user account](#create-aws-root-user-account)   
Step 2: [Create admin user group, admin access policy and assign users](#create-an-admin-iam-user-group-and-its-access-policy)    
Step 3: [Create non-admin user groups, non-admin access policies and assign users](#create-non-admin-iam-user-groups-and-their-access-policies)       


# Create AWS root user account   
When you create a free tier personal AWS account, you first create a [root user account](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html) which should only be used to:   
+ Create or delete an AWS account
+ Enable MFA for the root user account
+ Create or delete access keys for the root user account
+ Change the password for the root user account
+ Transfer root user account ownership

The first four tasks to complete in your root user account are to: 
1. [Set up multi-factor authentication for your root user account](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html#enable-virt-mfa-for-root).  
2. [Delete or inactivate your root user access key](https://docs.aws.amazon.com/accounts/latest/reference/root-user-access-key.html), as you should not use your root user account for everyday AWS tasks. This can be controlled in the top right corner via **Root user account -> Security credentials -> Access management -> Access keys** in the AIM control panel or using `aws iam delete-access-key --access-key-id EXAMPLEACCESSID` in CloudShell.  
3. [Enable AWS billing alerts and create an AWS billing alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html).  
    + Navigate to the **AWS Billing console** and tick both **Receive Free Tier Usage Alerts** and **Receive Billing Alerts** and save preferences.  
    + Change the region to **US East (N. Virginia)** via `export AWS_REGION=us-east-1` in CloudShell. Billing metric data is stored in the **US East (N. Virginia)** region.  
    + Navigate to the **CloudWatch console** and create an alarm for **Billing -> Total estimated charge**. Link your alarm to a subscription topic (supported by an AWS messaging service). Confirm your topic subscription via email. An alert should now appear in **CloudWatch -> Alarms -> All alarms** as shown below.  

![](./figures/successful_billing_alert.png)  

4. Set your default region via `export AWS_REGION=ap-southeast-2` and check your AWS configuration via `aws configure list` in CloudShell.   
</br>


# Create an admin IAM user group and its access policy           
[AWS recommends the creation of managed policies rather than inline policies to control user access to AWS resources.](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html) Managed policies can be attached to multiple users or user groups and governance is controlled around maintaining a central library of AWS policies. Policy changes automatically apply for all associated users or user groups. Inline policies should only be used when you want to maintain strict one-to-one relationships between a policy and a single AWS identity. 

![](./figures/aws_iam_policies.svg)  

The final task to complete in your root user account is to:
1. Create a new user group named `admin` using **Access management -> User groups** in the IAM console or via `aws iam create-group --group-name admin` in CloudShell.  
2. Create an admin access policy named `admin_access` via **Access management -> Policies -> Create policy** and input the following code into the JSON editor. The condition `"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}` restricts all AWS resource access to only the Sydney region. Because AWS account management resources like IAM, AWS Cost Explorer, CloudTrail and CloudShell can only be accessed via the global region i.e. default `us-east-1` region, we also need to explicitly allow access to these resources. AWS documentation about the latter step can be accessed [here](https://docs.aws.amazon.com/cloudshell/latest/userguide/sec-auth-with-identities.html).  

    <details><summary>JSON code</summary><p>  
    
    ```json  
    {
    "Version": "2012-10-17",
    "Statement": 
        [
            {
                "Sid": "UseAustralianResourcesOnly",
                "Effect": "Allow",
                "Action": "*",
                "Resource": "*",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}}
            },

            {
                "Sid": "UseConsole",
                "Effect": "Allow",
                "Action": [
                    "iam:*",
                    "cloudshell:*",
                    "access-analyzer:*",  
                    "aws-portal:ViewUsage",
                    "aws-portal:ViewBilling",
                    "aws-portal:ViewAccount"
                    ],
                "Resource": "*",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
            }
        ]
    }
    ```  
    </p></details>

3. Assign the `admin_access` policy to your previously created `admin` user group through **Access management -> User groups -> admin -> Permissions -> Add permissions -> admin_access** or `aws iam attach-group-policy --group-name admin --policy-arn <admin-access-arn>` in CloudShell.   
4. Create a new user named `admin-<name>` using `Access management -> Users -> Add user`, select **Password - AWS Management Console access** under AWS access type and add to the `admin` user group.  Alternatively, use `aws iam create-user --user-name admin-<name>`, then `aws iam create-login-profile --user-name admin-<name> --password <password>` and then `aws iam add-user-to-group --group-name admin --user-name admin-<name>` in CloudShell.    
5. [Activate admin user access to the AWS Billing console](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/control-access-billing.html) by navigating to **Account -> IAM User and Role Access to Billing Information -> Edit -> Activate IAM Access**.  
6. Log into your AWS account as `admin-<name>` and test your admin user group IAM policy by checking that you can access **IAM**, **CloudShell** and **Cost Explorer** via `us-east-1` but can only launch an EC2 instance from `ap-southeast-2`.   

You can now log into your administrator account to create more IAM user groups, access policies and users and access other cloud resources.    

>**Note** 
>  Access policies should follow the principle of least privilege, where users are given the minimal level of access privileges required for task completion. As a result, for non-admin user groups, applying JSON policy settings using `"Resource": "*"` or `"Action": "*"` is discouraged.   
</br>   


# Create non-admin IAM user groups and their access policies        
Log in via your `admin-<name>` IAM account to create more user groups. You can use the IAM console or CloudShell to create:    

+ An `engineer` user group for individuals with read and `write access to all S3 and Glue resources.   
+ A `analyst` user group for individuals with read access to the `source` S3 bucket and read and write access to specific folders in the `projects` S3 bucket. This user group also has read and write access to all EC2 instances, Sagemaker, Lambda and ECS.     

| Role type | Role | Engineer | Analyst |    
| --------- | ---- | -------- | ------- |     
| Platform Ops | View IAM policies | :heavy_check_mark: | :heavy_check_mark: |    
| Platform Ops | Create IAM policies | :x: | :x: |   
| Platform Ops | Manage MFA | :heavy_check_mark: | :heavy_check_mark: |    
| Platform Ops | Manage access keys | :heavy_check_mark: | :heavy_check_mark: |     
| Platform Ops | Access AWS CodeCommit | :heavy_check_mark: | :heavy_check_mark: |     
| Engineering | Access AWS Glue | :heavy_check_mark: | :x: |    
| Engineering | Create S3 buckets | :heavy_check_mark: | :x: |    
| Engineering | Create and edit S3 bucket policies | :heavy_check_mark: | :x: |     
| Engineering | Read objects inside `source` S3 bucket | :heavy_check_mark: | :heavy_check_mark: |    
| Engineering | Write objects inside `source` S3 bucket | :heavy_check_mark: | :x: |    
| Analysis | Read objects inside `projects` S3 bucket | :heavy_check_mark: | :heavy_check_mark: |    
| Analysis | Write objects inside `source` S3 bucket | :heavy_check_mark: | :heavy_check_mark: |   
| Analysis | Access AWS Sagemaker | :heavy_check_mark: | :heavy_check_mark: |     
| Cloud compute | Access EC2 instances | :heavy_check_mark: | :heavy_check_mark: |     
| Cloud compute | Access AWS Lambda | :heavy_check_mark: | :heavy_check_mark: |     
| Containerisation | Create AWS ECS images | :heavy_check_mark: | :heavy_check_mark: |     
</br>   

To create an `engineer` user group:  
1. Create a new user group named `engineer` using **Access management -> User groups** in the IAM console or via `aws iam create-group --group-name engineer` in CloudShell.   
2. Create an engineer access policy named `engineer_access` via **Access management -> Policies -> Create policy** and input the following code into the JSON editor.    

    <details><summary>JSON code</summary><p>  

    ```json  
    {
    "Version": "2012-10-17",
    "Statement": 
        [
            {
                "Sid": "UseEngineeringResources",
                "Effect": "Allow",
                "Action": [
                    "ec2:*",
                    "ecs:*",
                    "lambda:*",
                    "glue:*",
                    "cloudshell:*",
                    "s3:*"
                    ],
                "Resource": "*",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}}
            },

            {
                "Sid": "DenyUnencryptedS3ObjectUploads",
                "Effect": "Deny",
                "Action": "s3:PutObject",
                "Resource": "arn:aws:s3:::*",
                "Condition": {"Null": {"s3:x-amz-server-side-encryption": true}}
            },
        
            {
                "Sid": "UseConsole",
                "Effect": "Allow",
                "Action": [
                    "cloudshell:*",
                    "iam:PassRole",
                    "aws-portal:ViewUsage",
                    "aws-portal:ViewBilling",
                    "aws-portal:ViewAccount",
                    "iam:GetAccountPasswordPolicy",
                    "iam:ListMFADevices",
                    "iam:ListPolicies"
                    ],
                "Resource": "*",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
            },
            
            {
                "Sid": "ManageOwnMFADevice",
                "Effect": "Allow",
                "Action": [
                    "iam:CreateVirtualMFADevice",
                    "iam:DeleteVirtualMFADevice",
                    "iam:ListMFADevices",  
                    "iam:EnableMFADevice",
                    "iam:DeactivateMFADevice",
                    "iam:ResyncMFADevice"
                    ], 
                "Resource": "arn:aws:iam::*:mfa/${aws:username}",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
            },

            {
                "Sid": "ManageOwnSecurityCredentials",
                "Effect": "Allow",
                "Action": [
                    "iam:CreateServiceSpecificCredential",
                    "iam:DeleteServiceSpecificCredential",
                    "iam:ListServiceSpecificCredentials",
                    "iam:ResetServiceSpecificCredential",
                    "iam:UpdateServiceSpecificCredential",
                    "iam:CreateAccessKey",
                    "iam:DeleteAccessKey",
                    "iam:ListAccessKeys",
                    "iam:UpdateAccessKey",
                    "iam:GetAccessKeyLastUsed"
                    "iam:ChangePassword",
                    "iam:GetUser"  
                    ],
                "Resource": "arn:aws:iam::*:user/${aws:username}",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
            }
        ]
    }
    ```  
    </p></details>   

3. Assign the `engineer_access` policy to the `engineer` user group through **Access management -> User groups -> admin -> Permissions -> Add permissions -> engineer_access** or `aws iam attach-group-policy --group-name engineer --policy-arn <engineer-access-arn>` in CloudShell.     
4. Create a new IAM user named `engineer-<name>` using `Access management -> Users -> Add user`, select **Password - AWS Management Console access** under AWS access type and add to the `engineer` user group.     

> **Note**   
> The action `iam:PassRole` is required for `s3:CreateJob` in the engineer access policy and [user security management](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_aws_my-sec-creds-self-manage.html) is also enabled.     
</br>   

To create an `analyst` user group:  
1. Create a new user group named `analyst` using **Access management -> User groups** in the IAM console or via `aws iam create-group --group-name analyst` in CloudShell.  
2. Create an analyst access policy via **Access management -> Policies -> Create policy** and input the following code into the JSON editor.   

    <details><summary>JSON code</summary><p>  
    
    ```json   
    {
    "Version": "2012-10-17",
    "Statement": 
        [
            {
                "Sid": "UseDataScienceResources",
                "Effect": "Allow",
                "Action": [
                    "ec2:*",
                    "ecs:*",
                    "lambda:*",
                    "sagemaker:*",
                    "cloudshell:*"
                    ],
                "Resource": "*",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}}
            },
            
            {
                "Sid": "AccessLimitedS3Resources",
                "Effect": "Allow",
                "Action": [
                    "s3:ListAllMyBuckets",
                    "s3:ListBucket",
                    "s3:ListBucketVersions",
                    "s3:GetBucketVersioning",
                    "s3:GetBucketPolicyStatus",
                    "s3:GetBucketPublicAccessBlock",
                    "s3:GetAccountPublicAccessBlock",
                    "s3:GetBucketAcl",
                    "s3:GetObjectAcl",
                    "s3:ListAccessPointsForObjectLambda",
                    "s3:ListBucketMultipartUploads",
                    "s3:ListAccessPoints",
                    "s3:GetAccessPoint",
                    "s3:ListJobs",
                    "s3:CreateJob",
                    "s3:ListStorageLensConfigurations",
                    "s3:PutStorageLensConfiguration",
                    "s3:ListMultipartUploadParts",
                    "s3:ListMultiRegionAccessPoints",
                    "s3:GetBucketLocation", 
                    "s3:GetBucketPolicy",
                    "s3:GetBucketLogging",
                    "s3:GetBucketNotification",
                    "s3:GetBucketLocation",
                    "s3:GetEncryptionConfiguration"
                    ],
                "Resource": "arn:aws:s3:::*", 
                "Condition": {"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}}
            },

            {
                "Sid": "DenyUnencryptedS3ObjectUploads",
                "Effect": "Deny",
                "Action": "s3:PutObject",
                "Resource": "arn:aws:s3:::*",
                "Condition": {"Null": {"s3:x-amz-server-side-encryption": true}}
            },

            {
                "Sid": "UseConsole",
                "Effect": "Allow",
                "Action": [
                    "cloudshell:*",
                    "iam:PassRole",
                    "aws-portal:ViewUsage",
                    "aws-portal:ViewBilling",
                    "aws-portal:ViewAccount",
                    "iam:GetAccountPasswordPolicy",
                    "iam:ListMFADevices",
                    "iam:ListPolicies"
                    ],
                "Resource": "*",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
            },
            
            {
                "Sid": "ManageOwnVirtualMFADevice",
                "Effect": "Allow",
                "Action": [
                    "iam:CreateVirtualMFADevice",
                    "iam:DeleteVirtualMFADevice",
                    "iam:DeactivateMFADevice",
                    "iam:EnableMFADevice",
                    "iam:ListMFADevices",
                    "iam:ResyncMFADevice"
                    ],
                "Resource": "arn:aws:iam::*:mfa/${aws:username}",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
            },

            {
                "Sid": "ManageOwnSecurityCredentials",
                "Effect": "Allow",
                "Action": [
                    "iam:CreateServiceSpecificCredential",
                    "iam:DeleteServiceSpecificCredential",
                    "iam:ListServiceSpecificCredentials",
                    "iam:ResetServiceSpecificCredential",
                    "iam:UpdateServiceSpecificCredential",
                    "iam:CreateAccessKey",
                    "iam:DeleteAccessKey",
                    "iam:ListAccessKeys",
                    "iam:UpdateAccessKey",
                    "iam:GetAccessKeyLastUsed",
                    "iam:ChangePassword",
                    "iam:GetUser"
                    ],
                "Resource": "arn:aws:iam::*:user/${aws:username}",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
            }
        ]
    }
    ```  
    </p></details>  

3. Assign the `analyst_access` policy to your previously created `analyst` user group through **Access management -> User groups -> admin -> Permissions -> Add permissions -> analyst_access**.        
4. Create a new IAM user named `analyst-<name>` using `Access management -> Users -> Add user`, select **Password - AWS Management Console access** under AWS access type and add to the `analyst` user group.  

>**Note**  
> AWS resource access for the `analyst` user group includes unrestricted access to EC2, Sagemaker, Lambda and ECS. Bucket access is managed using S3 bucket policies for individual users rather than IAM policies for the analyst user group.      
</br>      




# Test IAM and S3 bucket policies    

To test our IAM policies, we can also run the following tests in CloudShell from the `engineer-<name>` and `analyst-<name>` IAM user accounts.   

### From the `engineer-<name>` user account:   

| Test | Command line code | Status |  
| ---- | ----------------- | ------ |  
| Create object in existing location in source bucket (encrypted) | `echo "hello world" \| aws s3 cp - s3://<name>-source/landing_zone/hw_raw.txt --sse AES256` |  :heavy_check_mark: |   
| Create object in new location in source bucket (encrypted) | `echo "hello world" \| aws s3 cp - s3://<name>-source/test/hw_test.txt --sse AES256` | :heavy_check_mark: | 
| Create object in new location in source bucket (unencrypted) | `echo "hello world" \| aws s3 cp - s3://<name>-source/test/hw_test.txt --sse AES256` | :x: |  
| List folders and objects in source bucket |  `aws s3 ls s3://<name>-source/ --recursive` | :heavy_check_mark: |  
| Delete new folder and folder contents in source bucket | `aws s3 rm s3://<name>-source/test --recursive` | :heavy_check_mark: |     
| Create object in new location in projects bucket (encrypted) |  `echo "hello world" \| aws s3 cp - s3://<name>-projects/test/hw_test.txt --sse AES256` <br> `echo "hello world" \| aws s3 cp - s3://<name>-projects/nlp_classification/hw_test.txt --sse AES256` | :heavy_check_mark: |    
| Delete new folder and folder contents in projects bucket | `aws s3 rm s3://<name>-projects/test --recursive` | :heavy_check_mark: |     
| Create new S3 bucket | `aws s3 mb s3://<name>-test` | :heavy_check_mark: | 
| Encrypt new S3 bucket | `aws s3api put-bucket-encryption --bucket <name>-test --server-side-encryption-configuration '{"Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]}'` | :heavy_check_mark: |    
| List buckets | `aws s3api list-buckets --query "Buckets[].Name"` | :heavy_check_mark: | 
| Delete S3 bucket | `aws s3api delete-bucket --bucket <name>-test` | :heavy_check_mark: |       

### From the `analyst-<name>` user account:     

According to the S3 bucket policy, we expect the analyst to have read and write access to `arn:aws:s3:::<name>-projects/palmer_penguins` but not `arn:aws:s3:::<name>-projects/nlp_classification`.   

| Test | Command line code | Status |  
| ---- | ----------------- | ------ |    
| List folders and objects in source bucket |  `aws s3 ls s3://<name>-source/ --recursive` | :heavy_check_mark: |  
| Create object in existing location in source bucket (encrypted) |  `echo "hello world" \| aws s3 cp - s3://<name>-source/landing_zone/hw_copy.txt --sse AES256` | :x: | 
| List folders and objects in projects bucket |  `aws s3 ls s3://<name>-projects/ --recursive` | :heavy_check_mark: |   
| Copy object from source bucket to permitted projects bucket folder | `aws s3 cp s3://<name>-source/landing_zone/hw_raw.txt s3://<name>-projects/palmer_penguins/test/hw_test.txt --sse AES256 ` | :heavy_check_mark: |      
| Copy object from source bucket to non-permitted projects bucket folder | `aws s3 cp s3://<name>-source/landing_zone/hw_raw.txt s3://<name>-projects/nlp_classification/test/hw_test.txt --sse AES256 ` | :x: |    
| Delete new subfolder and subfolder contents from permitted projects bucket folder | `aws s3 rm s3://<name>-projects/palmer_penguins/test --recursive` | :heavy_check_mark: |  
| Delete permitted projects bucket folder | `aws s3 rm s3://<name>-projects/palmer_penguins` | :x: |   
| List buckets | `aws s3api list-buckets --query "Buckets[].Name"` | :heavy_check_mark: |   
| Create new S3 bucket | `aws s3 mb s3://<name>-test` | :x: |   
| Delete S3 bucket | `aws s3api delete-bucket --bucket <name>-projects` | :x: |     
</br>   


# Create and sync CodeCommit repository to SageMaker    
AWS offers the ability to sync SageMaker Studio and SageMaker notebook instances to cloud code repositories like [GitHub and AWS CodeCommit](https://appwrk.com/aws-codecommit-vs-github-which-will-shine-in-2022). The recommended method of syncing users to CodeCommit is through [Git credentials and HTTPS with CodeCommit](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_ssh-keys.html).  

To create and sync a CodeCommit repository:  
1. Log into AWS using an admin user account and navigate to the CodeCommit console and click **Create repository**. 
2. Name your CodeCommit repository as `sagemaker_tutorials` and optionally add a repository description. Click **Create**.   
3. Ensure that you have a local Git client or download one [here](https://git-scm.com/downloads). Use `git --version` in your local CLI to check that your Git version is version 1.7.9 or later.  
4. Generate [Git credentials](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-gc.html?icmpid=docs_acc_console_connect_np) in the IAM console for all users. To do this, first create an access key ID and secret access key for all users by clicking on **Users -> Security credentials -> Access keys -> Create access key**.                
5. In the IAM console, for all users, navigate to **Users -> Permissions -> Add permissions -> Attach existing policies directly** and attach **AWSCodeCommitPowerUser**. Click **Review** and **Add permissions**.    




