# PGP Decryption for Transfer Family

## Configuring your AWS Environment


### CloudShell - Automated Creation of IAM Roles and Lambda Layer
- Open up CloudShell within your AWS account. 
- Run this command to download the setupEnvironment.sh script and the required IAM policies: 
  
  `wget https://github.com/aws-samples/pgp-decryption-for-transfer-family/blob/main/setupEnvironment.sh https://github.com/aws-samples/pgp-decryption-for-transfer-family/blob/main/IAM_Policies.zip`
  
- Run this command to create the required IAM roles and Lambda layer:
  
  `./setupEnvironment.sh`
  

### Deploying Custom Transfer Family Identity Provider (OPTIONAL)
- NOTE: If you already have a Transfer Family server in place that you want to use, or if you don't want to use a custom IDP, you can ignore this step. However, this project does require that you have a Transfer Family server running within your AWS account, so if you don't currently have one, I'd recommend deploying this CloudFormation stack as it will create the Transfer Family server + all the required IAM policies for you.  
- Refer to this link for detailed instructions on deploying the Custom Transfer Family Identity Provider via CloudFormation: [https://aws.amazon.com/blogs/storage/enable-password-authentication-for-aws-transfer-family-using-aws-secrets-manager-updated/](url)


### Creating Transfer Family Server
- If you don't want to deploy the custom Transfer Family identity provider via CloudFormation mentioned in the above step, and don't have a currently up and running Transfer Family server, please refer to this link for instructions on how to create a new Transfer Family server: https://docs.aws.amazon.com/transfer/latest/userguide/getting-started.html


### Creating the Lambda Function
- On the AWS Console, navigate to Lambda -> Functions
- Click "Create function"
- Select "Author from scratch"
- Name your Lambda function (Example: AutomatedPGPDecryption)
- Select "Python 3.8" as the Runtime
- Select "x86_64" as the Architecture
- Select "Change default execution role"
  - Select "Use an existing role"
  - Search "PGPDecryptionLambdaExecutionRole" and select it
- Click "Create Function"
- After creating the function, paste in the Python code from the lambdaSource.py file hosted on this GitHub.
- Click "Deploy" to save your changes

#### Attaching Layer to Lambda Function
- Scroll to the bottom of your Lambda function and select "Add a layer"
- Select "Custom layers"
- Choose "python-gnupg" as the layer
- Select whichever version is present and click "Add"


#### Editing Default Lambda Timeout
- Within your Lambda function console, select "Configuration" and then "General Configuration"
- Click "Edit"
- Change the timeout time from 3 seconds -> 15 seconds


### Creating an S3 Bucket (POSSIBLY OPTIONAL)
- NOTE: This step is only optional if you already have an S3 bucket configured that you'd like to use. 
- If not, you will need to follow these steps to create a new S3 bucket 
- Navigate to the S3 console within the AWS console
- Click "Create bucket"
- Name your bucket (Example: pgp-decrypted-files)
- Leave all options as default, unless you have specific requirements to do otherwise
- Scroll down to the bottom and select "Create bucket"



### Creating Transfer Family Managed Workflow
- Navigate to the Transfer Family console within the AWS console
- Select "Workflows"
- Select "Create Workflow"
- Provide a brief description of the workflow (Example: Automate PGP Decryption)
#### Step 1: Copy to Archive
- Under "Nominal steps", select "Add step"
  - Select "Copy file"
  - Name the step (Example: copyToArchive)
  - Select destination bucket (Example: "pgp-decrypted-files")
  - For Destination key prefix, insert the following: "Archive/${transfer:UserName}/" 
  - Select "Next" and then "Create step"
  
#### Step 2: Tag as Archived  
- Under "Nominal steps", select "Add step" 
  - Select "Tag file"
  - Name the step (Example: tagAsArchived)
  - For file location, select "Tag the file created from previous step"
  - For Key enter: "Status"
  - For Value enter: "Archived"
  - Click "Next" and the "Create step"

#### Step 3: PGP Decryption
- Under "Nominal steps", select "Add step"
  - Select "Custom file-processing step"
  - Name the step (Example: PGP_Decryption)
  - For file location, select "Apply custom processing to the original source file"
  - For target, select the Lambda function we created in earlier steps (Example: AutomatedPGPDecryption)
  - For timeout, leave as default (60 seconds)
  - Click "Next" and "Create step"

#### Step 4: Delete Originally Uploaded File
- Under "Nominal steps", select "Add step"
  - Select "Delete file"
  - Name the step (Example: Delete_Original_File)
  - For file location, select "Delete the original source file"
  - Click "Next" and "Create step"


#### Managed Workflow Exception Handlers

#### Step 1: Copy to Failed Prefix
- Under "Exception handlers - optional", select "Add step"
- Select "Copy file"
- Name the step (Example: copyToFailedPrefix)
- Select destination bucket (Example: "pgp-decrypted-files")
- For Destination key prefix, insert the following: "FailedDecryption/${transfer:UserName}/" 
- Select "Next" and then "Create step"

#### Step 2: Tag as Failed  
- Under "Exception handlers - optional", select "Add step"
  - Select "Tag file"
  - Name the step (Example: tagAsFailed)
  - For file location, select "Tag the file created from previous step"
  - For Key enter: "Status"
  - For Value enter: "Failed Decryption"
  - Click "Next" and the "Create step"

#### Step 3: Delete Originally Uploaded File
- Under "Exception handlers - optional", select "Add step"
  - Select "Delete file"
  - Name the step (Example: Delete_Original_File)
  - For file location, select "Delete the original source file"
  - Click "Next" and "Create step"


#### Final Step for Managed Workflow Creation
- Select "Create Workflow"


### Attach Managed Workflow to Transfer Family Server
- On the Transfer Family console, select "Servers"
- Select your desired Transfer Family server
- Under "Additional details", select "Edit"
- Select the newly created Workflow (Example: Automate PGP Decryption)
- Select the newly created Managed workflow execution role (Example: PGPDecryptionManagedWorkflowRole)
- Select "Save"




## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

