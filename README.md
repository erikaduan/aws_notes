# AWS usage notes  

![](https://img.shields.io/badge/Language-Bash-blue) ![](https://img.shields.io/badge/Language-Python-blue)  

This repository contains AWS cloudshell (bash) code snippets for setting up a secure AWS environment. Code snippets are sourced from the [AWS Cookbook](https://github.com/sous-chef/aws) or from [official AWS documentation](https://docs.aws.amazon.com/index.html). Architectural patterns are also sourced from the [UK Ministry of Justice AWS Security Guidelines](https://security-guidance.service.justice.gov.uk/baseline-aws-accounts/#baseline-for-amazon-web-services-accounts).      

>**Note**  
> You are provided with a management console (i.e. GUI) or command line option to perform operations inside AWS. The command line interface, also called cloudshell, can be access at the top right panel via the ![](https://github.com/erikaduan/aws_notes/blob/main/figures/cloudshell_icon.svg) icon.  
</br>


# Step 1: Create AWS root user account   
When you create a free personal AWS account, you first create a [root user account](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html) which should only be used to:  
+ Create or delete an AWS account
+ Enable MFA on the AWS account root user 
+ Create or delete access keys for the root user 
+ Change the password for the root user
+ Secure the credentials for the root user
+ Transfer root user owner

The first four tasks to do in your root user account are to: 
1. [Set up multi-factor authentication for your AWS account root user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html#enable-virt-mfa-for-root).  
2. [Delete or switch your root user access key to inactive](https://docs.aws.amazon.com/accounts/latest/reference/root-user-access-key.html), as you should not use your root user account for everyday AWS tasks. This can be controlled in the top right corner via **Root user account -> Security credentials -> Access management -> Access keys** in the AIM control panel or using `aws iam delete-access-key --access-key-id EXAMPLEACCESSID` in cloudshell.  
3. [Enable AWS billing alerts and create an AWS billing alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html).  
    + Navigate to the **AWS Billing console** and tick both **Receive Free Tier Usage Alerts** and **Receive Billing Alerts** and save preferences.  
    + Change the region to **US East (N. Virginia)** via `export AWS_REGION=us-east-1` in cloudshell. Billing metric data is stored in the **US East (N. Virginia)** region.  
    +  Navigate to the **CloudWatch console** and create an alarm for **Billing -> Total estimated charge**. Link your alarm to a subscription topic following the instructions in the AWS document above. Confirm your topic subscription via email. An alert should now appear in **CloudWatch -> Alarms -> All alarms** as shown below.
    </br>      
    ![](https://github.com/erikaduan/aws_notes/blob/main/figures/successful_billing_alert.png)  

4. Set your default region via `export AWS_REGION=ap-southeast-2` and check your AWS configuration via `aws configure list` in cloudshell.    
</br>


# Step 2: Create multiple IAM user accounts via the root user account   
[AWS recommends the creation of managed policies rather than inline policies.](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html) Managed policies can be attached to multiple users or user groups and governance is controlled around maintaining a central library of AWS policies. Policy changes automatically apply for all associated users or user groups.   

Inline policies should only be used when you want to maintain strict one-to-one relationships between a policy and AWS identity.  

Use AWS managed policies to create three user groups: IAM administrators, data engineers and data scientists.   

</br>


# Step 3: Creating a S3 bucket and assign IAM users with different S3 permissions  
 