#  Enable SageMaker access    
Step 1: [Add SageMaker access policies to existing user groups](#add-sagemaker-access-policies-to-existing-user-groups)    
Step 2: [Test SageMaker access](#test-sagemaker-access)     

# Add SageMaker access policies to existing user groups  
Although all SageMaker actions are enabled for the **engineer** and **analyst** user groups via IAM, a separate SageMaker full access policy must also be attached to user groups, so that interconnected EC2, ECR, Glue, CodeCommit, SecretsManager etc. access permissions are also enabled for SageMaker users.  

To set up access for your `admin`, `engineer` and `analyst` user groups:  
1. In IAM, attach the pre-created access policy **AmazonSageMakerFullAccess** to **engineer** and **analyst** user groups via ***User groups -> Group name -> Permissions -> Add permissions -> Attach policies -> AmazonSageMakerFullAccess -> Add permissions***.       
2. In IAM, navigate to ***Roles -> Create role -> AWS service*** and select ***Sagemaker-Execution*** under ***Use cases for other AWS services*** and click ***Next***. Name the role **sagemaker_role**, make sure that **AmazonSageMakerFullAccess** is selected and click ***Create role***.    
3. Create a separate JSON policy to enforce SageMaker [data encryption](https://docs.aws.amazon.com/sagemaker/latest/dg/security_iam_id-based-policy-examples.html#sagemaker-condition-kms) during model training and model hyperparameter tuning. Navigate to ***AWS Key Management Service -> Create a Key -> Symmetric -> Encrypt and decrypt***, name the key **SageMakerEncryptionKey** and click ***Next***. Assign the key administrator to **admin_\<name>**, assign all SageMaker users access to it (for example **admin_\<name>**, **analyst_\<name>** and **engineer_\<name>**) and create the KMS key. Click on the key ID and copy its ARN. Navigate back to IAM and create an access policy named **sagemaker_encryption** via ***Access management -> Policies -> Create policy*** and input the JSON code below into the JSON editor. Attach this policy to your SageMaker user groups via ***User groups -> Group name -> Permissions -> Add permissions -> Attach policies***.         

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

If you navigate to Sagemaker and encounter an error message like **...no identity-based policy allows the sagemaker:ListDomains action**, check that you have `"Action": ["iam:GetRole", "iam:CreateServiceLinkedRole"]` enabled for all resources in your user group IAM access policies.   

# Test SageMaker access    
To test SageMaker access:   
1. Log in as **analyst_\<user>** and navigate to ***SageMaker -> Domains -> Create domain -> Quick setup***. A domain contains a separate Elastic File System (EFS) for local storage and a single domain can be assigned to multiple users (each user will receive a separate personal home directory within the same EFS for their scripts, notebooks, Git repositories and data files).  
2. Assign your test domain a **unique name** like **\<user>-sagemaker-test**, append your username after the default user profile name provided to create a **unique user profile**, select **sagemaker_role** as the ***Execution role***, disable SageMaker Canvas permissions and click ***Create role***.    
3. After several minutes, your SageMaker domain will be created. Navigate to ***SageMaker -> Domains***, click on your **\<user>-sagemaker-test** domain and the ***Launch -> Studio*** button next to your user profile to launch SageMaker Studio.    
4. [Delete your test domain](https://docs.aws.amazon.com/sagemaker/latest/dg/gs-studio-delete-domain.html) using the console or CloudShell. When you delete a domain via the console, notebooks, data and git repositories stored inside your EFS are detached but not deleted. To delete the EFS along with the domain, set `HomeEfsFileSystem=Delete` in `RetentionPolicy` via CloudShell.    

| CLI command | action |     
| ----------- | ------ | 
| `aws --region ap-southeast-2 sagemaker list-domains` | List available SageMaker domains (included ones which failed to launch) |     
| `aws --region ap-southeast-2 sagemaker list-apps \--domain-id-equals <domain-id>` | List all applications associated with a domain |    
| `aws --region ap-southeast-2 sagemaker delete-app \--domain-id <domain-id> \--app-name <app-name> \--app-type <app-type> \--user-profile-name <user-profile-name>` | Delete each application associated with a domain |     
| `aws --region ap-southeast-2 sagemaker list-user-profiles \--domain-id-equals <domain-id>` | List all user profiles associated with a domain |    
| `aws --region ap-southeast-2 sagemaker delete-user-profile \--domain-id <domain-id> \--user-profile-name <user-profile-name>` | Delete each user profile associated a the domain |     
| `aws --region ap-southeast-2 sagemaker list-spaces \--domain-id <domain-id>` |  List all the shared spaces in adomain |   
| `aws --region ap-southeast-2 sagemaker delete-space \--domain-id <domain-id> \--space-name <space-name>` | Delete all shared spaces in a domain |    
| `aws --region ap-southeast-2 sagemaker delete-domain \--domain-id <domain-id> \--retention-policy HomeEfsFileSystem=Delete` | Delete the domain and associated EFS |    

