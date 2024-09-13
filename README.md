# AWS CodePipeline and CloudFormation Integration with Lambda and S3 Buckets

This README provides a step-by-step guide to implementing the following requirements using AWS services:

1. **Output CodePipeline and CloudFormation artifacts to separate S3 buckets**
2. **Sync with client S3 buckets using a prefix**
3. **Trigger Lambda functions from 2 client buckets**
4. **Invoke CodePipeline from Lambda**
5. **Create and delete CloudFormation stacks from Lambda**

## Prerequisites

Before starting, ensure the following resources and permissions are set up:
- AWS CLI installed and configured.
- Necessary IAM permissions for Lambda, S3, CodePipeline, and CloudFormation operations.
- An existing CodePipeline and CloudFormation template.

## Overview

This setup will:
1. Store **CodePipeline** and **CloudFormation** artifacts in separate S3 buckets.
2. Sync these artifacts with **client buckets** based on specified prefixes.
3. Automatically trigger a Lambda function whenever new objects are added to the client buckets.
4. The Lambda function will invoke **CodePipeline** and **CloudFormation** for further automation.
![image](https://github.com/user-attachments/assets/a09ff685-aef5-4642-a39d-440fff6860c7)


---

## 1. Output CodePipeline and CloudFormation Artifacts to Separate S3 Buckets

### Steps:

1. **Create two S3 buckets**: One for CodePipeline and one for CloudFormation artifacts.
    ```bash
    aws s3 mb s3://my-codepipeline-artifacts
    aws s3 mb s3://my-cloudformation-artifacts
    ```

2. **Configure CodePipeline to output to the CodePipeline bucket**:
    Modify your CodePipeline to include the `artifactStore` pointing to the S3 bucket:
    ```yaml
    artifactStore:
      type: S3
      location: my-codepipeline-artifacts
    ```

3. **Configure CloudFormation to store artifacts in the CloudFormation bucket**:
    Use the `aws cloudformation package` command to direct artifacts to your CloudFormation S3 bucket:
    ```bash
    aws cloudformation package --template-file template.yaml --s3-bucket my-cloudformation-artifacts --output-template-file packaged.yaml
    ```

---

## 2. Sync with Client S3 Buckets Using Prefix

To sync artifacts with client S3 buckets based on a specific prefix, create a Lambda function that listens for new S3 objects and triggers the sync.

1. **Set up client buckets**: Ensure that the client S3 buckets are ready.

2. **Lambda function to sync artifacts**:
    - Create a Lambda function that will be triggered by new objects added to the client buckets.
    - Use the S3 event notification system to trigger the Lambda.

    Example Lambda function for syncing:
    ```python
    import boto3

    s3 = boto3.client('s3')

    def lambda_handler(event, context):
        source_bucket = event['Records'][0]['s3']['bucket']['name']
        object_key = event['Records'][0]['s3']['object']['key']
        
        destination_bucket = "client-bucket"
        s3.copy_object(
            Bucket=destination_bucket,
            CopySource={'Bucket': source_bucket, 'Key': object_key},
            Key=f'prefix/{object_key}'
        )
        return {'status': 'synced'}
    ```

3. **Set up S3 bucket event notifications**:
    - Configure S3 events for the client buckets to trigger Lambda on object creation:
    ```bash
    aws s3api put-bucket-notification-configuration \
      --bucket client-bucket-1 \
      --notification-configuration file://notification.json
    ```

---

## 3. Trigger Lambda Function from 2 Client Buckets

1. **Configure both client S3 buckets** to trigger the same Lambda function:
    - In the S3 bucket settings, add event notifications for `s3:ObjectCreated:*` events and associate them with the Lambda ARN.

2. **S3 Event Notification configuration**:
    ```json
    {
      "LambdaFunctionConfigurations": [
        {
          "Id": "SyncWithLambda",
          "LambdaFunctionArn": "arn:aws:lambda:region:account-id:function:sync-function",
          "Events": ["s3:ObjectCreated:*"]
        }
      ]
    }
    ```

---

## 4. Call CodePipeline from Lambda

1. **Invoke CodePipeline from Lambda**:
    - Create a Lambda function that triggers the pipeline whenever it receives an event from the S3 bucket.
    
    Example Lambda function to trigger CodePipeline:
    ```python
    import boto3

    def lambda_handler(event, context):
        client = boto3.client('codepipeline')
        response = client.start_pipeline_execution(name='my-pipeline')
        return response
    ```

2. **Configure IAM permissions**:
    Ensure that the Lambda function has the appropriate permissions to invoke CodePipeline.

    Example policy:
    ```json
    {
      "Effect": "Allow",
      "Action": "codepipeline:StartPipelineExecution",
      "Resource": "arn:aws:codepipeline:region:account-id:my-pipeline"
    }
    ```

---

## 5. Call Create and Delete CloudFormation from Lambda

1. **Lambda Function to Create and Delete CloudFormation Stacks**:
    - Use `boto3` to interact with CloudFormation from Lambda.

    Example Lambda function to create a CloudFormation stack:
    ```python
    import boto3

    def create_stack():
        client = boto3.client('cloudformation')
        response = client.create_stack(
            StackName='my-stack',
            TemplateURL='https://s3.amazonaws.com/my-cloudformation-artifacts/template.yaml',
            Parameters=[
                {
                    'ParameterKey': 'KeyName',
                    'ParameterValue': 'my-key'
                }
            ]
        )
        return response
    ```

    Example Lambda function to delete a CloudFormation stack:
    ```python
    def delete_stack():
        client = boto3.client('cloudformation')
        response = client.delete_stack(StackName='my-stack')
        return response
    ```

2. **Configure Lambda for Create/Delete actions**:
    - The Lambda function should trigger based on specific conditions, such as object creation in the S3 bucket or manually via the AWS Console or an API.

    - Ensure that the Lambda function has the necessary IAM permissions to create and delete CloudFormation stacks:
    ```json
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:DeleteStack"
      ],
      "Resource": "*"
    }
    ```

---

## Conclusion

This setup leverages various AWS services to automate and streamline processes using S3, Lambda, CodePipeline, and CloudFormation. With this infrastructure, you can ensure a scalable, automated deployment and management of your resources.
