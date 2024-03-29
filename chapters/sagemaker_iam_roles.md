#  Enable SageMaker access    
Step 1: [Add SageMaker data encryption access policy to existing user groups](#add-sagemaker-data-encryption-access-policy-to-existing-user-groups)   
Step 2: [Enable SageMaker access to other AWS services]()    
Step 3: [Test SageMaker access](#test-sagemaker-access)    
Step 4 (Optional): [Set up Visual Studio Code access to SageMaker]()   


# Add SageMaker data encryption access policy to existing user groups  
Although all SageMaker actions are now enabled for the **admin**, **engineer** and **analyst** user groups via IAM, a separate SageMaker data encryption key and associated IAM policy can be generated and attached.  

To set up a separate SageMaker data encryption key and associated IAM policy:  
1. Log in as **admin-\<name>** and navigate to Key Management System (KMS) to create a key for SageMaker [data encryption](https://docs.aws.amazon.com/sagemaker/latest/dg/security_iam_id-based-policy-examples.html#sagemaker-condition-kms) during model training and model hyperparameter tuning. Access ***AWS Key Management Service -> Create a Key -> Symmetric -> Encrypt and decrypt***, name the key **SageMakerEncryptionKey** and click ***Next***. 
2. Assign the key administrator to **admin_\<name>**, assign all SageMaker users access to it (for example **admin-\<name>**, **analyst-\<name>** and **engineer-\<name>**) and create the KMS key. Click on the key ID and copy its ARN. 
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
    
    > **Note**
    > The following IAM policy creation and attachment step can only be performed by the **admin** user group. Creation of notebook instances and SageMaker domains (described in the next section below) should be possible for all **admin**, **engineer** and **analyst** user groups.

If you navigate to SageMaker and encounter an error message like **...no identity-based policy allows the sagemaker:ListDomains action**, check that you have the actions `"sagemaker:ListDomains"`, `"iam:GetRole"`, `"iam:CreateRole"`, `"iam:CreateServiceLinkedRole"` and `"iam:ListAttachedRolePolicies"` enabled for all your user groups requiring SageMaker access.


# Create SageMaker role with access to all S3 buckets
For SageMaker to function, it needs to be attached to a separate role that allows SageMaker access to other AWS services like S3, Glue, Lambda and etc. A role is different to an IAM policy for a user group as it assigns permissions for an AWS service rather than a person and all roles have a lifecycle rule i.e., permissions are deactivated following 12 hours. You can create a single SageMaker execution role and reuse it for future SageMaker domains or notebook instances.   

To create and attach a SageMaker role for the first time:  
1. Log in as **admin-\<name>** and navigate to ***Notebook -> Notebook instances -> Create notebook instance*** to create a new notebook instance. This is currently the easiest method of auto-generating a SageMaker full access policy with S3 permissions attached.     
2. Create a notebook instance name unique to your AWS account region like **\<team-name>-data-science**. Select an instance type (choose the default **ml.t3.medium** instance for standard projects) and the latest Amazon Linux image and Jupyter Lab version (Amazon Linux 2 and Jupyter Lab 3 as of Jan 2023).   
3. Under ***Permissions and encryption -> IAM role***, select ***Create a new role***. In the new window, make sure you have S3 bucket access ticked under Specific S3 buckets and enter **\<name>-source** and **\<name>-projects**. 
4. Click ***Create role*** and a new IAM role with a name beginning in **AmazonSageMaker-Execution-Role** should appear and be selected. 
5. Select the **SageMakerEncryptionKey** generated above under ***Encryption key - optional*** and click ***Create notebook instance***.  

Once your new notebook instance is created, you can navigate to ***IAM -> Roles*** and click on the new role with a name beginning in **AmazonSageMaker-Execution-Role** to check its properties. You should see two policies attached to this role: 1. **AmazonSageMakerFullAccess** and 2. a customer managed **AmazonSageMaker-ExecutionPolicy** which allows SageMaker Get, Put, Delete and List access in S3 buckets. Remember to select this **AmazonSageMaker-Execution-Role** role when creating new SageMaker Studio domains or notebook instances in the future.     


# Test SageMaker access    

To test SageMaker Studio access:   
1. Log in as **analyst_\<user>** and navigate to ***SageMaker -> Domains -> Create domain -> Quick setup***. A domain contains a separate Elastic File System (EFS) for local storage and a single domain can be assigned to multiple users (each user will receive a separate personal home directory within the same EFS for their scripts, notebooks, Git repositories and data files).  
2. Assign your test domain a **unique name** like **\<user>-sagemaker-access-test**, and add a unique user profile name like **\<first-name>-\<last-name>**, select your newly created **AmazonSageMaker-Execution-Role** as described above, disable SageMaker Canvas permissions and click ***Create role***.    
3. After several minutes, your SageMaker domain will be created. Navigate to ***SageMaker -> Domains***, click on your **\<user>-sagemaker-test** domain and the ***Launch -> Studio*** button next to your user profile to launch SageMaker Studio.    
4. Inside SageMaker Studio, select ***File -> New -> Notebook*** and select an EC2 instance type to run your data science docker image in. For example **Data Science 3.0** for image, **Python 3** for kernel, **ml.t3.medium** for instance type and no start-up script. Once your notebook kernel has loaded, run some test Python code in the first cell.  

    ```python 
    import numpy as np

    x = np.array([(1,2,3),(4,5,6)]) 
    print(x)
    ```

5. Once you have successfully run your Python code, click ***File -> Shut Down -> Shutdown All*** in order to exit from SageMaker Studio and shut down your data science image and the underlying Jupyter Server application. You can click on ***domain -> user profile*** to navgiate to the ***User Details*** page and and check that all applications listed under your user profile have been deleted. Manually delete applications if this has not occurred.       
6. [Delete your user profile and test domain](https://docs.aws.amazon.com/sagemaker/latest/dg/gs-studio-delete-domain.html) using the console or CloudShell. When you delete a domain via the console, notebooks, data and git repositories stored inside your EFS are detached but not deleted. To delete the EFS along with the domain, set `HomeEfsFileSystem=Delete` in `RetentionPolicy` via CloudShell. All user profiles associated with a domain must be deleted first, before the domain can be deleted.      
<br> 

The following bash code can also be used instead of clicking through the SageMaker console.   

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
<br> 

To test SageMaker notebook instance access:
1. Remain logged in as **analyst_\<user>** and navigate to ***Notebook -> Notebook instances -> Create notebook instance***.
2. Assign your instance a unique name like **\<user>-notebook-access-test**. Select an instance type (choose the default **ml.t3.medium** instance for standard projects) and the latest Amazon Linux image and Jupyter Lab version (Amazon Linux 2 and Jupyter Lab 3 as of Jan 2023). Select the **AmazonSageMaker-Execution-Role** with S3 access enabled as previously created and enable root access to users.
3. (Optional) Select the **SageMakerEncryptionKey** generated above under ***Encryption key - optional*** and click ***Create notebook instance***.
4. Once the status of the newly created notebook instance is **InService**, click ***Open JupyterLab*** next to the **\<user>-notebook-access-test** notebook instance. This opens a new Jupyter Lab environment.
5. Create a new R notebook via ***File -> New -> Notebook*** and select your kernel as the **R** kernel to run the following test R code.

    ```r
    # Code to list all pre-installed packages in the R kernel  
    str(allPackage <- installed.packages(.Library))
    allPackage [, c(1,3:5)]
    ```

6. Once you have successfully run your R code, exit from your notebook instance via ***File -> Shut Down*** and then click ***Stop*** on your notebook instance page to stop its associated EC2 instance.  
7. Delete your test notebook instance by clicking on the notebook instance name and then the ***Delete*** button.  
<br>  

The following bash code can also be used instead of clicking through the SageMaker console.   

| CLI command | action |
| ----------- | ------ |
| `aws --region ap-southeast-2 sagemaker list-notebook-instances` | List available SageMaker notebook instances (included ones which failed to launch) |
| `aws sagemaker start-notebook-instance --notebook-instance-name <notebook-name>` | Start a specific notebook instance |
| `aws sagemaker delete-notebook-instance --notebook-instance-name <notebook-name>` | Delete a specific notebook instance |
<br> 


# Set up Visual Studio Code access to SageMaker   
Some users may find it easier to access the AWS CLI through their local IDE (for example Visual Studio Code), especially if they use scripts to build and interact with AWS services. Setting up a local connection to the AWS CLI also enables users to copy objects from their local environment into S3 buckets using bash.  

To set up local CloudShell (AWS CLI) access:  
1. Install [the latest version of the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#cliv2-windows-install) on your local desktop. Confirm that the installation has been successful by navigating to your command prompt window and typing `aws --version`. This should output the version of the AWS CLI that you have just installed locally.   
2. Log in AWS as **admin-\<name>** and navigate to ***Users -> analyst_\<name> -> Security credentials -> Access keys -> Create access key***. This will take you to a screen with a ***Download .csv file*** button that displays an **access key ID** and **secret Access key**. Keep both of these records private and save the `.csv` file somewhere where it cannot be commited to a public code repository.  
3. In an organisation, the admin then sends this information to user **analyst_\<name>** through a secure process. It is good practice to deactivate old access keys and create and use new ones semi-regularly to decrease the chances of old access keys being misappropriated by third parties.   
3. Log in AWS as **analyst-\<name>** and open your local command prompt window, for example via ***Search -> Command Prompt -> Right click -> Open as administrator*** for Windows users. Type `aws configure` and enter the **access key ID**, **secret access key** and enter `ap-southeast-2` for the default region. Enter `JSON` for the default output format.        
4. Following this, you have now gained programatic access to your AWS resources via your local command prompt. Test this by entering `aws s3 ls` in the command prompt, which should output the list of S3 buckets available to **analyst-\<name>**.  
5. You can now upload local datasets directly into your S3 bucket by downloading a dataset, for example the [R Palmer penguins dataset](https://github.com/allisonhorst/palmerpenguins/blob/main/inst/extdata/penguins.csv), into a local file director and then using the following bash template.  

```
cd .\data 

aws s3 cp palmer_penguins.csv s3://<name>-projects/palmer_penguins_analysis/palmer_penguins.csv --sse AES256 

ls aws s3 ls s3://<name>-projects --recursive
```
<br> 

Congratulations! You have now set up SageMaker access for your **analyst** and **engineer** user groups and enabled local command prompt access to your AWS cloud account and services for your user **analyst-\<name>**.   


# Other resources    
+ Blog post on [SageMaker usage via the AWS CLI](https://hackmd.io/@AWS-TeamB-HSPF-2/B1Vx5qFgd)    
+ Instructions on [how to set up local computer access to the AWS CLI](https://adamtheautomator.com/upload-file-to-s3/)    