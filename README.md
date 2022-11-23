# AWS usage notes - a comprehensive guide for AWS newbies    

![](https://img.shields.io/badge/Language-Bash-blue) ![](https://img.shields.io/badge/Language-Python-blue) ![](https://img.shields.io/badge/Language-R-blue)    

| Icon | AWS resource | Topic | Description |
| :--- | :----------- | :---- | :---------- |
| :cowboy_hat_face: | Identity & Access Management (IAM) | [Create new AWS user groups, users and access policies](./chapters/iam_roles_and_access_policies.md) | Create root user account, user groups, users, roles and policies for platform administrator, data engineer and data scientist users. |     
| :bucket: | S3 bucket | Manage S3 bucket permissions | Finetune object storage permissions separately using S3 bucket permissions. |    
| :notebook_with_decorative_cover: | Sagemaker | [Manage Python and R environments](./chapters/sagemaker_environment_management.md) | Manage R and Python environments and packages in Sagemaker. |   


## Tips on learning to use AWS            
+ AWS provides a management console (i.e. GUI) and command line options to perform operations. The command line interface, also called CloudShell, can be accessed at the top right panel via the **>_** icon.   
+ AWS resources can also be accessed using the Python software development kit (SDK) [`boto3`](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html). For data transformations, use the [`awswrangler`](https://aws-sdk-pandas.readthedocs.io/en/stable/) Python SDK.      
+ Learn to write shell Bash scripts to execute processes through the cloud command line interface.    

## Other resources    
+ Data Science on AWS [github repository](https://github.com/data-science-on-aws/data-science-on-aws) - contains code snippets for setting up AWS services and creating data science workflows.  