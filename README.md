# AWS usage notes  

![](https://img.shields.io/badge/Language-Bash-blue) ![](https://img.shields.io/badge/Language-Python-blue)  

Step 1: [Create root user account](#create-root-user-account)  
Step 2: [Create an IAM admin user and user group via the root user account](#create-an-iam-admin-user-and-user-group-via-the-root-user-account)  
Step 3: [Create S3 buckets](#create-s3-buckets)  
Step 4: [Create user groups with different S3 bucket access policies](#create-more-user-groups-via-the-iam-admin-user-account)  
Step 5: [Launch an EC2 instance](#launch-an-ec2-instance)  

This repository contains AWS console instructions and command line interface bash snippets for setting up a secure AWS environment. Code snippets are sourced from the [AWS Cookbook](https://github.com/sous-chef/aws) or from [official AWS documentation](https://docs.aws.amazon.com/index.html). Architectural patterns are also sourced from the [UK Ministry of Justice AWS Security Guidelines](https://security-guidance.service.justice.gov.uk/baseline-aws-accounts/#baseline-for-amazon-web-services-accounts) and [Statistics Canada AWS resouces](https://github.com/StatCan/daaas).  

>**Note** 
> You are provided with both a management console (i.e. GUI) or command line option to perform operations inside AWS. The command line interface, also called CloudShell, can be accessed at the top right panel via the |>_| icon.  
</br>


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

![](/figures/successful_billing_alert.png)  

4. Set your default region via `export AWS_REGION=ap-southeast-2` and check your AWS configuration via `aws configure list` in CloudShell.   
</br>


# Create an IAM admin user and user group via the root user account   
[AWS recommends the creation of managed policies rather than inline policies to control user access to AWS resources.](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html) Managed policies can be attached to multiple users or user groups and governance is controlled around maintaining a central library of AWS policies. Policy changes automatically apply for all associated users or user groups. Inline policies should only be used when you want to maintain strict one-to-one relationships between a policy and a single AWS identity. 

![](/figures/aws_iam_policies.svg)  

The final task to complete in your root user account is to:
1. Create a new user group named `admin` using **Access management -> User groups** in the IAM console or via `aws iam create-group --group-name admin` in CloudShell.  
2. Create an admin access policy named `admin_access` via **Access management -> Policies -> Create policy** and input the following code into the JSON editor. The condition `"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}` restricts all AWS resource access to only the Sydney region. Because AWS account management resources like IAM, AWS Cost Explorer, CloudTrail and CloudShell can only be accessed via the global region i.e. default `us-east-1` region, we also need to explicitly allow access to these resources. AWS documentation about the latter step can be accessed [here](https://docs.aws.amazon.com/cloudshell/latest/userguide/sec-auth-with-identities.html).  

    ```
    {
    "Version": "2012-10-17",
    "Statement": 
        [
            {
                "Sid": "AccessAustralianResourcesOnly",
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

3. Assign the `admin_access` policy to your previously created `admin` user group through **Access management -> User groups -> admin -> Permissions -> Add permissions -> admin_access** or `aws iam attach-group-policy --group-name admin --policy-arn <admin_access arn>` in CloudShell.   
4. Create a new IAM user named `admin_<name>` using `Access management -> Users -> Add user`, select **Password - AWS Management Console access** under AWS access type and add to the `admin` user group.  Alternatively, use `aws iam create-user --user-name admin_<name>`, then `aws iam create-login-profile --user-name admin_<name> --password <password>` and then `aws iam add-user-to-group --group-name admin --user-name admin_<name>` in CloudShell.    
5. [Activate IAM admin user access to the AWS Billing console](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/control-access-billing.html) by navigating to **Account -> IAM User and Role Access to Billing Information -> Edit -> Activate IAM Access**.  

You can now log into your IAM administrator account to create more IAM users, user groups, access policies and cloud resources.  

**Note:** Access policies should follow the principle of least privilege, where users are given the minimal level of access privileges required for task completion. As a result, for non-admin user groups, applying JSON policy settings using `"Resource": "*"` or `"Action": "*"` are discouraged.    
**Note:** You can test whether your `admin` JSON policy has been correctly applied by checking that you can access **IAM**, **CloudShell** and **Cost Explorer** via `us-east-1`, but can only launch an EC2 instance from `ap-southeast-2` as user `admin_<name>`.   
</br> 


# Create S3 buckets      
We will first create two S3 bucket resources, one for data ingress and egress from AWS and one for data access inside AWS.  

To create the S3 buckets, log into AWS as an IAM admin user:  
1. [Create a S3 bucket using the S3 console](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html) and click **Create bucket**.  
2. This takes you to a new page where you need to assign a unique bucket name i.e. `<name>-landing-zone`, confirm your AWS region as `ap-southeast-2`, keep ACLs disabled under **Object Ownership**, block all public access under **Block Public Access settings for this bucket** , enable data object versioning under **Bucket Versioning** and enable server-side encryption using Amazon S3-managed keys under **Default encryption**.   
3. Click **Create bucket**. Navigate to the **Properties** tab to locate the S3 bucket Amazon Resource Name (ARN) i.e. `arn:aws:s3:::<name>-landing-zone`. 
4. Repeat this process and assign an unique name to a second S3 bucket i.e. `<name>-analysis`. This bucket will then have the Amazon Resource Name `arn:aws:s3:::<name>-analysis`.  

When ACLs are disabled, the bucket owner i.e. `admin_<name>` automatically owns and has full control over every object in the bucket. The IAM admin user can then create separate bucket policies for different user groups.   

**Note:** It is essential to block all public access settings to S3 buckets.   
**Note:** You can also manage data access at the level of S3 bucket folders as described [here](https://docs.aws.amazon.com/AmazonS3/latest/userguide/walkthrough1.html#walkthrough-background1).    
</br>   


# Create user groups with different S3 bucket access policies    
Log in via your `admin_<name>` IAM account to create more user groups. You can use the IAM console or CloudShell to create:  
+ An `engineer` user group for individuals with `GET` and `PUT` access to all S3 and Glue resources.   
+ A `analyst` user group for individuals with `GET` access to specific S3 buckets and `GET` and `PUT` access to EC2 instances, Sagemaker, ECS and EKS.   

We ideally want to separate data engineering and data analysis work components by S3 bucket access policies. When accessing AWS from a private external environment, we would also want to restrict data migration tasks to a limited group of individuals i.e. the `engineer` user group.   
![](/figures/aws_s3_access.svg)  

To create an `engineer` user group:  
1. Create a new user group named `engineer` using **Access management -> User groups** in the IAM console or via `aws iam create-group --group-name engineer` in CloudShell.  
2. Create an engineer access policy named `engineer_access` via **Access management -> Policies -> Create policy** and input the following code into the JSON editor. AWS resource access for the `engineer` user group includes unrestricted access to Glue and unrestricted access to `arn:aws:s3:::<name>-landing-zone` and `arn:aws:s3:::<name>-analysis` S3 buckets. Resource and object ARNS need to be specified for `s3:GetObject` and `s3:PutObject` actions and `iam:PassRole` is required for `s3:CreateJob`. The JSON policy settings for enabling user security settings management are listed [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_aws_my-sec-creds-self-manage.html).  

    ```
    {
    "Version": "2012-10-17",
    "Statement": 
        [
            {
                "Sid": "AccessAWSGlue",
                "Effect": "Allow",
                "Action": "glue:*",
                "Resource": "*",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}}
            },
        
            {
                "Sid": "AccessAllS3Settings",
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
                    "s3:CreateAccessPoint",
                    "s3:ListJobs",
                    "s3:CreateJob",
                    "s3:ListStorageLensConfigurations",
                    "s3:PutStorageLensConfiguration",
                    "s3:ListMultipartUploadParts",
                    "s3:ListMultiRegionAccessPoints",
                    "s3:GetBucketPolicy",
                    "s3:GetBucketLogging",
                    "s3:GetBucketNotification",
                    "s3:GetEncryptionConfiguration",
                    ],
                "Resource": "*", 
                "Condition": {"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}}
            },

            {
                "Sid": "AccessAllS3Buckets",
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:GetObjectVersion", 
                    "s3:GetObjectVersionAttributes",
                    "s3:GetObjectAttributes",
                    "s3:PutObject",
                    "s3:DeleteObject"
                    ],
                "Resource": [
                    "arn:aws:s3:::erika-analysis",
                    "arn:aws:s3:::erika-landing-zone",
                    "arn:aws:s3:::erika-analysis/*",
                    "arn:aws:s3:::erika-landing-zone/*"
                    ], 
                "Condition": {"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}}
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
                    "iam:DeleteVirtualMFADevice"
                    ],
                "Resource": "arn:aws:iam::*:mfa/${aws:username}",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
            },

            {
                "Sid": "ManageOwnSecurityCredentials",
                "Effect": "Allow",
                "Action": [
                    "iam:DeactivateMFADevice",
                    "iam:EnableMFADevice",
                    "iam:ListMFADevices",
                    "iam:ResyncMFADevice",
                    "iam:CreateServiceSpecificCredential",
                    "iam:DeleteServiceSpecificCredential",
                    "iam:ListServiceSpecificCredentials",
                    "iam:ResetServiceSpecificCredential",
                    "iam:UpdateServiceSpecificCredential",
                    "iam:CreateAccessKey",
                    "iam:DeleteAccessKey",
                    "iam:ListAccessKeys",
                    "iam:UpdateAccessKey",
                    "iam:ChangePassword",
                    "iam:GetUser"
                    ],
                "Resource": "arn:aws:iam::*:user/${aws:username}",
                "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
            }
        ]
    }
    ```

    3. Assign the `engineer_access` policy to your previously created `engineer` user group through **Access management -> User groups -> admin -> Permissions -> Add permissions -> engineer_access** or `aws iam attach-group-policy --group-name engineer --policy-arn <engineer_access arn>` in CloudShell.   
    4. Create a new IAM user named `engineer_<name>` using `Access management -> Users -> Add user`, select **Password - AWS Management Console access** under AWS access type and add to the `engineer` user group.     
    5. Test that the `engineer_access` policy has been correctly applied. Log into your AWS account as `engineer_<name>` and confirm that you can upload and delete data objects and create S3 bucket access points for both `arn:aws:s3:::erika-landing-zone` and `arn:aws:s3:::erika-analysis`.    

To create an `analyst` user group:  
1. Create a new user group named `analyst` using **Access management -> User groups** in the IAM console or via `aws iam create-group --group-name analyst` in CloudShell. 
2. Create an analyst access policy via **Access management -> Policies -> Create policy** and input the following code into the JSON editor.  

    ```
    ```

**Note:** Refer to [this resource](https://medium.com/tensult/aws-policies-with-examples-8340661d35e9) for more AWS access policy examples.   
</br>


# Launch an EC2 instance    
+ EC2 instance types
    + General purpose instances (small to medium datasets)
    + Compute optimized instances (high CPU for modelling etc.)
    + Memory optimized instances (large datasets need to be processed in-memory)
    + Accelerated computing instances (GPUs?)
    + Storage optimized instances  (data storage applications)

+ Elastic load balancing is based on a region and is used for EC2 scaling. Decoupled architecture from the front end and the back end. 

Elastic Load Balancing (distributes incoming internet traffic) and Amazon EC2 Auto Scaling are separate services, they work together to help ensure that applications running in Amazon EC2 can provide high performance and availability  

# Other notes 
AWS EKS Amazon Elastic Kubernetes Service
Test docket container 
container orchestration tools 
Test serverless compute via AWS lambda functions 