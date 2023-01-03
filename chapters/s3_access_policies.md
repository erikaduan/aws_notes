# Manage S3 bucket permissions  

Step 1: [Create S3 buckets and S3 bucket policies](#create-s3-buckets-and-s3-bucket-policies)    
Step 2: [Test S3 bucket policies](#test-s3-bucket-policies)   

# Change log   
<br>  

# Create S3 buckets and S3 bucket policies   
[Access to S3 resources](https://aws.amazon.com/blogs/security/iam-policies-and-bucket-policies-and-acls-oh-my-controlling-access-to-s3-resources/) can be controlled using multiple methods. In general, IAM policies allow or restrict S3 bucket resource permissions for user groups whereas S3 bucket policies are attached to specific S3 buckets and enable fine tuning of S3 bucket content permissions at the [principal](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html) i.e. individual user level.       

S3 bucket permissions can be set at the bucket, root level or folder level.  

In our case, we would like to create two S3 bucket resources, one named `source` and one named `projects`.    

+ The `source` bucket will contain a `landing_zone` folder and `bronze_layer` folder, with data analysts granted GET access to the `bronze_layer` folder.   
+ The `projects` bucket will contain multiple folders, each hosting a separate project. Analysts have GET and PUT access to their own project folders.   

![](../figures/aws_s3_access.svg)  

To create the `source` S3 bucket:    
1. Log into AWS using a data engineer user account and navigate to the S3 console and click **Create bucket**.    
2. This takes you to a new page where you need to assign a unique bucket name i.e. `<name>-source`, confirm your AWS region as `ap-southeast-2`, keep ACLs disabled under **Object Ownership**, block all public access under **Block Public Access settings for this bucket**, enable data object versioning under **Bucket Versioning** and enable server-side encryption using Amazon S3-managed keys under **Default encryption**.    
3. Confirm bucket creation by clicking **Create bucket** again.  
4. Click on the newly created bucket and navigate to the **Properties** tab to locate its Amazon Resource Name (ARN) i.e. `arn:aws:s3:::<name>_source`.  
5. Navigate back to the **Object** tab and click **Create folder** to create the `landing_zone` and `bronze_layer` folders. Remember to enable server-side encryption using Amazon S3-managed keys.    
6. Navigate to the **Permissions** tab and scroll down to the [bucket policy](https://docs.aws.amazon.com/AmazonS3/latest/userguide/add-bucket-policy.html) section and click **edit**. Note that only the bucket owner can associate a policy with the S3 bucket.   
7. Create the `source` S3 bucket policy and input the following code into the JSON editor.     

    ```json    
    {
    "Version": "2012-10-17",
    "Statement": 
        [
            {
                "Sid": "RestrictAnalystAccess",
                "Effect": "Allow",  
                "Principal": {
                    "AWS": "arn:aws:iam::<aws-account-id>:user/analyst_<name>"
                },  
                "Action": [
                    "s3:GetObject",
                    "s3:GetObjectVersion",
                    "s3:GetObjectVersionAttributes",
                    "s3:GetObjectAttributes"
                    ],
                "Resource": [
                    "arn:aws:s3:::<name>-source",
                    "arn:aws:s3:::<name>-source/*"
                    ]
            }
        ]
    }
    ```     
    > **Note** 
    > Individual AWS user accounts have to be specified under `Principle` in the S3 bucket policy i.e. no wildcard user accounts allowed.     

To create the `projects` S3 bucket:  
1. Remain logged into your data engineer user account and navigate to the S3 console and repeat steps 1 to 4 above i.e. create a new S3 bucket named `<name>_projects` with the same bucket settings.
2. Navigate back to the **Object** tab and click **Create folder** to create the `palmer_penguins` and `nlp_classification` folders. Remember to enable server-side encryption using Amazon S3-managed keys.    
3. Navigate to the **Permissions** tab and scroll down to the [bucket policy](https://docs.aws.amazon.com/AmazonS3/latest/userguide/add-bucket-policy.html) section and click **edit**. Note that only the bucket owner can associate a policy with the S3 bucket.   
4. Create the `projects` S3 bucket policy and input the following code into the JSON editor. This policy only allows the user `analyst-<name>` to have read and write access to the `palmer_penguins` and not `nlp_classification` folder.         
    
    ```json    
    {
    "Version": "2012-10-17",
    "Statement": 
        [
            {
                "Sid": "AnalystPalmerPenguinAccess",
                "Effect": "Allow", 
                "Principal": {
                    "AWS": "arn:aws:iam::<aws-account-id>:user/analyst_<name>"
                },  
                "Action": [
                    "s3:GetObject",
                    "s3:GetObjectVersion", 
                    "s3:GetObjectVersionAttributes",
                    "s3:GetObjectAttributes",
                    "s3:PutObject"
                    ],
                "Resource": [
                    "arn:aws:s3:::<name>-projects/palmer_penguins",
                    "arn:aws:s3:::<name>-projects/palmer_penguins/*"
                    ]
            },

            {
                "Sid": "AnalystPalmerPenguinDelete",
                "Effect": "Allow",  
                "Principal": {
                    "AWS": "arn:aws:iam::<aws-account-id>:user/analyst_<name>"
                },  
                "Action": "s3:DeleteObject", 
                "Resource": "arn:aws:s3:::<name>-projects/palmer_penguins/*"
            }
        ]
    }
    ```    

    > **Note** 
    > Individual AWS user accounts have to be specified under `Principle` in the S3 bucket policy i.e. no wildcard user accounts allowed.   
  

 When IAM and S3 bucket policies both exist for a user, access is determined as the least-privilege union of all the user permissions. Always avoid using `"Principal": "*"` with an `allow` effect in S3 bucket policies, as this will enable public access to your AWS resources. Also avoid setting up S3 bucket permissions using S3 ACLs as this is a legacy permissions maintenance system.      
</br> 

# Test S3 bucket policies  


