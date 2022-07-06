# AWS usage notes  

![](https://img.shields.io/badge/Language-Bash-blue) ![](https://img.shields.io/badge/Language-Python-blue)  

This repository contains AWS cloudshell (bash) code snippets for setting up a secure AWS environment. Code snippets are sourced from the [AWS Cookbook](https://github.com/sous-chef/aws) or from [official AWS documentation](https://docs.aws.amazon.com/index.html). Architectural patterns are also sourced from the [UK Ministry of Justice AWS Security Guidelines](https://security-guidance.service.justice.gov.uk/baseline-aws-accounts/#baseline-for-amazon-web-services-accounts).      

**Note:** You are provided with a management console (i.e. GUI) or command line option to perform operations inside AWS. The command line interface, also called cloudshell, can be access at the top right panel via the ![](https://github.com/erikaduan/aws_notes/blob/main/figures/cloudshell_icon.svg) icon.  
</br>


# Step 1: Create AWS root user account   
When you create a free personal AWS account, you first create a [root user account](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html) which should only be used to:  
+ Create or delete an AWS account
+ Enable MFA on the AWS account root user 
+ Create or delete access keys for the root user 
+ Change the password for the root user
+ Secure the credentials for the root user
+ Transfer root user owner

The first four tasks to complete in your root user account are to: 
1. [Set up multi-factor authentication for your AWS account root user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html#enable-virt-mfa-for-root).  
2. [Delete or switch your root user access key to inactive](https://docs.aws.amazon.com/accounts/latest/reference/root-user-access-key.html), as you should not use your root user account for everyday AWS tasks. This can be controlled in the top right corner via **Root user account -> Security credentials -> Access management -> Access keys** in the AIM control panel or using `aws iam delete-access-key --access-key-id EXAMPLEACCESSID` in cloudshell.  
3. [Enable AWS billing alerts and create an AWS billing alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html).  
    + Navigate to the **AWS Billing console** and tick both **Receive Free Tier Usage Alerts** and **Receive Billing Alerts** and save preferences.  
    + Change the region to **US East (N. Virginia)** via `export AWS_REGION=us-east-1` in cloudshell. Billing metric data is stored in the **US East (N. Virginia)** region.  
    +  Navigate to the **CloudWatch console** and create an alarm for **Billing -> Total estimated charge**. Link your alarm to a subscription topic following the instructions in the AWS document above. Confirm your topic subscription via email. An alert should now appear in **CloudWatch -> Alarms -> All alarms** as shown below.  

![](https://github.com/erikaduan/aws_notes/blob/main/figures/successful_billing_alert.png)  

4. Set your default region via `export AWS_REGION=ap-southeast-2` and check your AWS configuration via `aws configure list` in cloudshell.   
</br>


# Step 2: Create an IAM admin user and user group via the root user account   
[AWS recommends the creation of managed policies rather than inline policies to control user access to AWS resources.](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html) Managed policies can be attached to multiple users or user groups and governance is controlled around maintaining a central library of AWS policies. Policy changes automatically apply for all associated users or user groups. Inline policies should only be used when you want to maintain strict one-to-one relationships between a policy and AWS identity.  
</br>
![](https://github.com/erikaduan/aws_notes/blob/main/figures/aws_iam_policies.svg)  

The final task to complete in your root user account is to: 
1. Create a new user group named `admin` using **Access management -> User groups** in the IAM console or via `aws iam create-group --group-name admin` in cloudshell.    
2. Create an admin access policy via **Access management -> Policies -> Create policy** and input the following code into the JSON editor. The condition `"StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}` restricts all AWS resource access to only the Sydney region.    
    ```
    {
    "Version": "2012-10-17",
    "Statement": 
        [
            {
                "Effect": "Allow",
                "Action": "*",
                "Resource": "*",
                "Condition": 
                {
                    "StringEquals": {"aws:RequestedRegion": "ap-southeast-2"}
                }
            }
        ]
    }
    ```
3. Name this policy `admin_access` and assign it to your previously created `admin` user group through **Access management -> User groups -> admin -> Permissions -> Add permissions -> admin_access** or `aws iam attach-group-policy --group-name admin --policy-arn <admin_access arn>` in cloudshell.   
4. Create a new IAM user named `admin_<name>` using `Access management -> Users -> Add user`, select **Password - AWS Management Console access** under AWS access type and add to `admin` user group.  Alternatively, use `aws iam create-user --user-name admin_<name>`, then `aws iam create-login-profile --user-name admin_<name> --password <password>` and then `aws iam add-user-to-group --group-name admin --user-name admin_<name>` in cloudshell.    

You can now log into your IAM administrator account and create more IAM users, user groups, access policies and cloud resources.   
</br> 


# Step 3: Create power user and analyst users and user groups via the IAM admin user account  
You can now use the IAM console or cloudshell to create:  
+ A `poweruser` user group for individuals with read and write access to all AWS resources.  
+ An `analyst` user group for individuals with read and write access to specific AWS resources i.e. specific S3 buckets.  

**Note:** Refer to [this resource](https://medium.com/tensult/aws-policies-with-examples-8340661d35e9) for more AWS access policy examples.   
</br>


# Step 4: Creating a S3 bucket and assign IAM users with different S3 permissions  
 