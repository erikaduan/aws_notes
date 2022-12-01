#  Enable Sagemaker access    

Step 1: [Create Sagemaker IAM access policies](#create-sagemaker-iam-access-policies)    
Step 2:  
Step 3:    


# Create Sagemaker IAM access policies
By default, AWS users do not have permission to access AWS Sagemaker and this need to be enables using IAM access policies.    

To set up access for your `admin`, `engineer` and `analyst` user groups:  
1. Attach the pre-created access policy [`AmazonSageMakerFullAccess`](https://us-east-1.console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonSageMakerFullAccess) to your user groups of interest via **User groups -> Group name -> Permissions -> Add permissions -> Attach policies**.       
2. Create an access policy named `sagemaker_domain_access` via **Access management -> Policies -> Create policy** and input the JSON code below into the JSON editor. Attach this policy to your user groups of interest via **User groups -> Group name -> Permissions -> Add permissions -> Attach policies**.   

    ```json  
        {
        "Version": "2012-10-17",
        "Statement": 
            [
                {
                    "Sid": "AllowSagemakerDomainAccess",
                    "Effect": "Allow",
                    "Action": "sagemaker:*",
                    "Resource": [
                        "arn:aws:sagemaker:*:*:domain/*",
                        "arn:aws:sagemaker:*:*:user-profile/*",
                        "arn:aws:sagemaker:*:*:app/*",
                        "arn:aws:sagemaker:*:*:flow-definition/*"
                        ]
                },

                {
                    "Sid": "EnableServiceCatalogAccess",
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetRole",
                        "servicecatalog:*"
                        ],
                    "Resource": "*"    
                }
            ]
        }
    ```    

3. Enforce [input data encryption](https://docs.aws.amazon.com/sagemaker/latest/dg/security_iam_id-based-policy-examples.html#sagemaker-condition-kms) by navigating to AWS Key Management Service. Create a symmetric key to encrypt and decrypt data and assign trusted users as its key administrators. Apply this key to all accounts requiring Sagemaker access and create the key. Click on the key name to locate and copy its ARN.   
4. Navigate back to IAM and create an access policy named `sagemaker_encryption` via **Access management -> Policies -> Create policy** and input the JSON code below into the JSON editor. Attach this policy to your user groups of interest via **User groups -> Group name -> Permissions -> Add permissions -> Attach policies**.       

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
                    "Condition": {"ArnEquals": {"sagemaker:VolumeKmsKey": "arn:aws:<kms_arn>"}}
                }
            ]
        }
    ```    

> **Note**    
> If you navigate to Sagemaker and encounter an error message like `...no identity-based policy allows the sagemaker:ListDomains action`, check that you have created and attached `sagemaker_domain_access` to your user group access permissions.         






