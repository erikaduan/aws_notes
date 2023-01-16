#  Enable SageMaker access    
Step 1: [Add SageMaker data encryption access policy to existing user groups](#add-sagemaker-data-encryption-access-policy-to-existing-user-groups)   
Step 2: [Enable SageMaker access to other AWS services]()    
Step 3: [Test SageMaker access](#test-sagemaker-access)    
Step 4 (Optional): [Set up Visual Studio Code access to SageMaker]()   

# Add SageMaker data encryption access policy to existing user groups  
Although all SageMaker actions are now enabled for the **admin**, **engineer** and **analyst** user groups via IAM, a separate SageMaker data encryption key and associated IAM policy can be generated and attached.  

To set up a separate SageMaker data encryption key and associated IAM policy:  
1. Navigate to Key Management System (KMS) to create a key for SageMaker [data encryption](https://docs.aws.amazon.com/sagemaker/latest/dg/security_iam_id-based-policy-examples.html#sagemaker-condition-kms) during model training and model hyperparameter tuning. Access ***AWS Key Management Service -> Create a Key -> Symmetric -> Encrypt and decrypt***, name the key **SageMakerEncryptionKey** and click ***Next***. 
2. Assign the key administrator to **admin_\<name>**, assign all SageMaker users access to it (for example **admin_\<name>**, **analyst_\<name>** and **engineer_\<name>**) and create the KMS key. Click on the key ID and copy its ARN. 
3. Navigate back to IAM and create an access policy named **sagemaker_encryption** via ***Access management -> Policies -> Create policy*** and input the JSON code below into the JSON editor. Attach this policy to **admin**, **analyst** and **engineer** user groups via ***User groups -> Group name -> Permissions -> Add permissions -> Attach policies***.         

    ```json  
        {
        "Version": "2012-10-17",
        "Statement": 
            [
                {
                    "Sid": "EnforceEncryption",
                    "Effect": "Allow",
                    "Action": [
                        "sagemaker:CreateTrainingJob",
                        "sagemaker:CreateHyperParameterTuningJob",
                        "sagemaker:CreateLabelingJob",
                        "sagemaker:CreateFlowDefiniton"
                        ],
                    "Resource": "*",
                    "Condition": {"ArnEquals": {"sagemaker:VolumeKmsKey": "<kms-arn>"}}
                }
            ]
        }
    ```    

If you navigate to SageMaker and encounter an error message like **...no identity-based policy allows the sagemaker:ListDomains action**, check that you have the actions `"sagemaker:ListDomains"`, `"iam:GetRole"`, `"iam:CreateRole"`, `"iam:CreateServiceLinkedRole"` and `"iam:ListAttachedRolePolicies"` enabled for all your user groups requiring SageMaker access.    

# Create SageMaker role with access to all S3 buckets  
For SageMaker to function, it needs to be attached to a separate role that allows SageMaker access to other AWS services like S3, Glue, Lambda and etc. A role is different to an IAM policy for a user group as it assigns permissions for an AWS service rather than a person and all roles have a lifecycle rule i.e., permissions are deactivated following 12 hours.  

To set up a SageMaker role for the first time:  
1. Log in as **admin-\<name>** and navigate to ***Notebook -> Notebook instances -> Create notebook instance*** to create a new notebook instance. This is currently the easiest method of auto-generating a SageMaker full access policy with S3 permissions attached.     
2. Create a notebook instance name unique to your AWS account region like **\<team-name>-data-science**. Select an instance type (choose the default **ml.t3.medium** instance for standard projects) and the latest Amazon Linux image and Jupyter Lab version (Amazon Linux 2 and Jupyter Lab 3 as of Jan 2023).   
3. Under ***Permissions and encryption -> IAM role***, select ***Create a new role***. In the new window, make sure you have S3 bucket access ticked under Specific S3 buckets and enter **\<name>-source** and **\<name>-projects**. 
4. Click ***Create role*** and a new IAM role with a name beginning in **AmazonSageMaker-Execution-Role** should appear and be selected. 
5. Select the **SageMakerEncryptionKey** generated above under ***Encryption key - optional*** and click ***Create notebook instance***.  

Once your new notebook instance is created, you can navigate to ***IAM -> Roles*** and click on the new role with a name beginning in **AmazonSageMaker-Execution-Role** to check its properties. You should see two policies attached to this role: 1. **AmazonSageMakerFullAccess** and 2. a customer managed **AmazonSageMaker-ExecutionPolicy** which allows SageMaker Get, Put, Delete and List access in S3 buckets. Remember to select this **AmazonSageMaker-Execution-Role** role when creating new SageMaker Studio domains or notebook instances in the future.     


# Test SageMaker access    
To test SageMaker Studio access:   
1. Log in as **analyst_\<user>** and navigate to ***SageMaker -> Domains -> Create domain -> Quick setup***. A domain contains a separate Elastic File System (EFS) for local storage and a single domain can be assigned to multiple users (each user will receive a separate personal home directory within the same EFS for their scripts, notebooks, Git repositories and data files).  
2. Assign your test domain a **unique name** like **\<user>-sagemaker-test**, and add a unique user profile name like **\<first-name>-\<last-name>**, select your newly created **AmazonSageMaker-Execution-Role** as described above, disable SageMaker Canvas permissions and click ***Create role***.    
3. After several minutes, your SageMaker domain will be created. Navigate to ***SageMaker -> Domains***, click on your **\<user>-sagemaker-test** domain and the ***Launch -> Studio*** button next to your user profile to launch SageMaker Studio.    
4. Test that SageMaker Domain works!
5. [Delete your test domain](https://docs.aws.amazon.com/sagemaker/latest/dg/gs-studio-delete-domain.html) using the console or CloudShell. When you delete a domain via the console, notebooks, data and git repositories stored inside your EFS are detached but not deleted. To delete the EFS along with the domain, set `HomeEfsFileSystem=Delete` in `RetentionPolicy` via CloudShell.    

| CLI command | action |     
| ----------- | ------ | 
| `aws --region ap-southeast-2 sagemaker list-domains` | List available SageMaker domains (included ones which failed to launch) |     
| `aws --region ap-southeast-2 sagemaker list-apps \--domain-id-equals <domain-id>` | List all applications associated with a domain |    
| `aws --region ap-southeast-2 sagemaker delete-app \--domain-id <domain-id> \--app-name <app-name> \--app-type <app-type> \--user-profile-name <user-profile-name>` | Delete each application associated with a domain |     
| `aws --region ap-southeast-2 sagemaker list-user-profiles \--domain-id-equals <domain-id>` | List all user profiles associated with a domain |    
| `aws --region ap-southeast-2 sagemaker delete-user-profile \--domain-id <domain-id> \--user-profile-name <user-profile-name>` | Delete each user profile associated a the domain |     
| `aws --region ap-southeast-2 sagemaker list-spaces \--domain-id <domain-id>` |  List all the shared spaces in adomain |   
| `aws --region ap-southeast-2 sagemaker delete-space \--domain-id <domain-id> \--space-name <space-name>` | Delete all shared spaces in a domain |    
| `aws --region ap-southeast-2 sagemaker delete-domain \--domain-id <domain-id> \--retention-policy HomeEfsFileSystem=Delete` | Finally delete the domain and its associated EFS |    

# Set up Visual Studio Code access to SageMaker  