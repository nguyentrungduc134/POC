# AWS CodePipeline and CodeDeploy with Lambda Integration - README

## Overview
This document outlines the process to set up a CI/CD pipeline using **AWS CodePipeline**, **CodeDeploy**, **Lambda**, and **S3**. It includes:
1. Outputting **CodePipeline** and **CloudFormation** artifacts to separate S3 buckets.
2. Syncing artifacts with client S3 buckets using specific prefixes.
3. Triggering a Lambda function from two client buckets.
4. Using the Lambda function to call **CodeDeploy** for application deployment.
5. Using the Lambda function to call **CodeDeploy** for infrastructure deployment.
![image](https://github.com/user-attachments/assets/6eb55658-2689-4272-b6d5-7694ba9b0285)

## Prerequisites
- **AWS CLI** installed and configured with proper permissions.
- **AWS IAM** roles with necessary permissions for **CodePipeline**, **CodeDeploy**, **Lambda**, and **S3**.
- Two S3 buckets for storing artifacts.
- Existing CodePipeline, CodeDeploy application, and infrastructure setup.

## Steps to Implement

### 1. Output CodePipeline and CloudFormation Artifacts to Separate S3 Buckets
- Create two S3 buckets for storing CodePipeline and CloudFormation artifacts (e.g., `codepipeline-artifacts` and `cloudformation-artifacts`).
- Update your CodePipeline to specify the S3 bucket for storing artifacts in the pipeline configuration:
  ```yaml
  ArtifactStore:
    Type: S3
    Location: codepipeline-artifacts
  ```
- In the CloudFormation template, specify the S3 bucket for artifact storage:
  ```yaml
  Parameters:
    ArtifactBucket:
      Type: String
      Default: cloudformation-artifacts
  ```

### 2. Sync with Client Buckets Using Prefix
- Use an S3 event to trigger synchronization whenever a new artifact is uploaded to your `codepipeline-artifacts` bucket.
- Configure S3 bucket notification with the prefix to filter specific objects and call the Lambda function:
  ```yaml
  NotificationConfiguration:
    LambdaConfigurations:
      - Event: s3:ObjectCreated:*
        Filter:
          S3Key:
            Rules:
              - Name: prefix
                Value: "client-artifacts/"
        LambdaFunctionArn: arn:aws:lambda:REGION:ACCOUNT_ID:function:SyncLambdaFunction
  ```
- The Lambda function script for syncing artifacts could be something like:
  ```python
  import boto3

  s3_client = boto3.client('s3')

  def lambda_handler(event, context):
      source_bucket = 'codepipeline-artifacts'
      destination_bucket = 'client-artifacts'
      object_key = event['Records'][0]['s3']['object']['key']

      s3_client.copy_object(
          Bucket=destination_bucket,
          CopySource={'Bucket': source_bucket, 'Key': object_key},
          Key=object_key
      )
  ```

### 3. Trigger Lambda Function from 2 Client Buckets
- Create two client buckets that will trigger a Lambda function on the upload of new artifacts.
- Set up **S3 bucket notifications** to invoke the Lambda function on `s3:ObjectCreated` events for these buckets:
  ```yaml
  NotificationConfiguration:
    LambdaConfigurations:
      - Event: s3:ObjectCreated:*
        LambdaFunctionArn: arn:aws:lambda:REGION:ACCOUNT_ID:function:DeployLambdaFunction
  ```
- This Lambda function will handle invoking **CodeDeploy** for both application and infrastructure deployments.

### 4. Call CodeDeploy for Application from Lambda
- Inside the Lambda function, use the **Boto3** library to call **CodeDeploy**:
  ```python
  import boto3

  codedeploy_client = boto3.client('codedeploy')

  def lambda_handler(event, context):
      response = codedeploy_client.create_deployment(
          applicationName='MyApplication',
          deploymentGroupName='MyApplicationDeploymentGroup',
          revision={
              'revisionType': 'S3',
              's3Location': {
                  'bucket': 'codepipeline-artifacts',
                  'key': 'application-artifact.zip',
                  'bundleType': 'zip'
              }
          },
          deploymentConfigName='CodeDeployDefault.AllAtOnce',
          description='Deploying application from Lambda'
      )
  ```
- Ensure that the Lambda function's IAM role has the necessary permissions to call **CodeDeploy**.

### 5. Call CodeDeploy for Infrastructure from Lambda
- Modify the Lambda function to include a separate **CodeDeploy** call for the infrastructure:
  ```python
  def lambda_handler(event, context):
      # CodeDeploy for application
      codedeploy_client.create_deployment(
          applicationName='MyApplication',
          deploymentGroupName='MyApplicationDeploymentGroup',
          revision={
              'revisionType': 'S3',
              's3Location': {
                  'bucket': 'codepipeline-artifacts',
                  'key': 'application-artifact.zip',
                  'bundleType': 'zip'
              }
          },
          deploymentConfigName='CodeDeployDefault.AllAtOnce',
          description='Deploying application from Lambda'
      )

      # CodeDeploy for infrastructure
      codedeploy_client.create_deployment(
          applicationName='MyInfrastructure',
          deploymentGroupName='MyInfrastructureDeploymentGroup',
          revision={
              'revisionType': 'S3',
              's3Location': {
                  'bucket': 'cloudformation-artifacts',
                  'key': 'infrastructure-artifact.zip',
                  'bundleType': 'zip'
              }
          },
          deploymentConfigName='CodeDeployDefault.AllAtOnce',
          description='Deploying infrastructure from Lambda'
      )
  ```

## Architecture Diagram
*(Include the diagram in your README. Assume it shows S3 buckets, Lambda functions, CodePipeline, and CodeDeploy integrations.)*

## Summary
This setup ensures:
- Separate storage of CodePipeline and CloudFormation artifacts in dedicated S3 buckets.
- Synchronization of artifacts with client buckets using S3 events and Lambda functions.
- Automatic triggering of deployments via CodeDeploy using Lambda functions.
- Centralized deployment control for both application and infrastructure components.

## Additional Notes
- Ensure proper IAM policies are attached to your Lambda functions to allow access to S3, CodeDeploy, and other necessary services.
- Test the entire flow in a staging environment before deploying it in production.
