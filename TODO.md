TODO 

4. Navigate to **Access Points** and create a new access point named `<name>-landing-zone-access`. Select internet under **Network Origin**, block all public access under **Block Public Access settings for this Access Point** and record the Access Point ARN.    
5. Create a S3 bucket policy for `<name>-landing-zone-access` that allows analyst users to also access `arn:aws:s3:::<name>-landing-zone` though its access point.   

    <details><summary>JSON code</summary><p>  

    ```json
    {
        "Version": "2012-10-17",
        "Statement" : 
        [
            {
                "Effect": "Allow",
                "Principal" : {"AWS": "arn:aws:iam::<aws-account-id>:user/analyst-<name>"},
                "Action" : "s3:*",
                "Resource" : "arn:aws:s3:ap-southeast-2:<aws-account-id>:accesspoint/<name>-landing-zone-access",  
                "Condition": {"StringEquals": {"s3:DataAccessPointAccount": "<aws-account-id>"}}
            }
        ]
    }
    ```
    </p></details>

6. Repeat this process and assign an unique name to a second S3 bucket i.e. `<name>-analysis` and second access point i.e. `<name>-analysis-access`. The bucket will then have the Amazon Resource Name `arn:aws:s3:::<name>-analysis` and a separate Access Point ARN and access point bucket policy doe analyst users.      

When ACLs are disabled, the bucket owner i.e. `admin-<name>` automatically owns and has full control over every object in the bucket. The IAM admin user can then create separate bucket policies for different user groups.  

>**Note**  
> It is essential to block all public access settings to S3 buckets. This needs to be managed when you create a S3 bucket resource and when you create its IAM or bucket access policies. Avoid using `"Principal": "*"` at all costs in JSON policies with an `allow` effect, as this will enable public access to your AWS resources.    

>**Note**  
> You can also manage data access at the level of S3 bucket folders as described [here](https://docs.aws.amazon.com/AmazonS3/latest/userguide/walkthrough1.html#walkthrough-background1).    
</br>   

# Testing  

3. Test that the `engineer_access` policy has been correctly applied. Log into your AWS account as `engineer-<name>` and confirm that you can access CloudShell, upload and delete data objects and create S3 bucket access points.  
4. Upload a test dataset in `arn:aws:s3:::<name>-landing-zone/raw`. Confirm that you can copy the test dataset with encryption into `arn:aws:s3:::<name>-analysis` using `aws s3 cp s3://<name>-landing-zone/raw/test.csv s3://<name>-analysis/test.csv --sse AES256` in CloudShell.  
5. Confirm that unencrypted data uploads are denied i.e. `aws s3 cp s3://<name>-landing-zone/raw/test.csv s3://<name>-analysis/test.csv` fails in CloudShell.   

## Create an `analyst_access` IAM policy  

3. Confirm that the `analyst_access` policy has been correctly applied. Log into your AWS account as `analyst-<name>` and confirm that you can open but not create new encrypted data objects or folders in `arn:aws:s3:::<name>-landing-zone`. Confirm that you can create and delete encrypted folders in `arn:aws:s3:::<name>-analysis`.  
4. Confirm that # TODO  

## CLI tests for engineer and analyst user groups   
To test our IAM policies, we can also run the following tests in CloudShell from the `engineer-<name>` and `analyst-<name>` IAM user accounts.  

| CLI test | engineer | analyst |  
| ------------------------------------------------------------------------------------ | ------------------ | --- |   
| `echo "hi" \| aws s3 cp - s3://<name>-landing-zone/test/hw.txt --sse AES256` | :heavy_check_mark: | :x: |   
| `aws s3 ls s3://<name>-landing-zone/test/`  | :heavy_check_mark:  | :heavy_check_mark: |     
| `echo "hi" \| aws s3 cp - s3://<name>-analysis/test/hw.txt --sse AES256` | :heavy_check_mark: | :x: |     
| `aws s3 ls s3://<name>-analysis/test/` | :heavy_check_mark:  | :heavy_check_mark: |     
| `aws s3 cp s3://<name>-landing-zone/test/hw.txt s3://<name>-analysis/test/hw_copy.txt --sse AES256` | :heavy_check_mark: | Failed |     
| `aws s3 cp s3://<name>-analysis/test/hw.txt s3://<name>-landing-zone/test/hw_copy.txt --sse AES256` | :heavy_check_mark: | :x: |   
| `aws s3 cp s3://<name>-analysis/test/hw.txt s3://<name>-analysis/test_copy/hw_copy.txt --sse AES256` |  | Failed |     
| `aws s3 rm s3://<name>-landing-zone/test/hw_copy.txt` | :heavy_check_mark:  | :x: |   
| `aws s3 rm s3://<name>-analysis/test/hw_copy.txt` | :heavy_check_mark:  | :heavy_check_mark: |   
| `aws s3 cp s3://<name>-landing-zone/test/hw.txt s3://<name>-analysis/test/hw.txt` | :x: | :x: |    

>**Note**  
> You can also test the generation of JSON policies using the [AWS policy generator wizard](https://awspolicygen.s3.amazonaws.com/policygen.html) and read more details about creating IAM policies with different S3 permissions [here](https://aws.amazon.com/blogs/security/writing-iam-policies-grant-access-to-user-specific-folders-in-an-amazon-s3-bucket/).   
</br>


# Launch an EC2 instance    
An EC2 instance is a virtual server which supports our AWS resources. CloudShell can be viewed as a free version of a general purpose EC2 instance that allows users to run AWS CLI commands to manage infrastructure and automate tasks.       

+ EC2 instance types
    + General purpose instances (small to medium datasets)
    + Compute optimized instances (high CPU for modelling etc.)
    + Memory optimized instances (large datasets need to be processed in-memory)
    + Accelerated computing instances (GPUs?)
    + Storage optimized instances  (data storage applications)

+ Recommendation to run an EC2 instance across at least two avaiability zones within a region (availability zone is a proper subset of a region).  
+ Elastic load balancing is based on a region and is used for EC2 scaling. Decoupled architecture from the front end and the back end.   

Elastic Load Balancing (distributes incoming internet traffic) and Amazon EC2 Auto Scaling are separate services, they work together to help ensure that applications running in Amazon EC2 can provide high performance and availability  


# Configure a Sagemaker domain for the analyst user group     



#TODO 

# Manage AWS credentials as an analyst user   
Once the user has been created, see Managing access keys to learn how to create and retrieve the keys used to authenticate the user.

If you have the AWS CLI installed, then you can use the aws configure command to configure your credentials file:

aws configure
Alternatively, you can create the credentials file yourself. By default, its location is ~/.aws/credentials. At a minimum, the credentials file should specify the access key and secret access key. In this example, the key and secret key for the account are specified the default profile:

[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
You may also want to add a default region to the AWS configuration file, which is located by default at ~/.aws/config:

[default]
region=us-east-1
Alternatively, you can pass a region_name when creating clients and resources.

You have now configured credentials for the default profile as well as a default region to use when creating connections. See Configuration for in-depth configuration sources and options.
#TODO you can set this up to test `boto3`.  


# Other notes 
AWS EKS Amazon Elastic Kubernetes Service
Test docket container 
container orchestration tools 
Test serverless compute via AWS lambda functions 
https://github.com/data-science-on-aws/data-science-on-aws  