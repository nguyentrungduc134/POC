# README: Setting Up CodePipeline and CloudFormation Artifacts Management  
<!-- x-release-please-start-version -->
  ```
    Version : 'x.x.x'
  ```
<!-- x-release-please-end -->
This document outlines the process of automating deployments using AWS CodePipeline, Lambda, and S3. It covers artifact management, synchronizing client buckets, DNS configuration, and Lambda-triggered CodePipeline deployments.

## Overview

1. **Output CodePipeline and CloudFormation artifacts to separate S3 buckets.**
2. **Copy source artifacts to client buckets using a prefix and date suffix via Lambda.** (Estimated Time: 5 hours)
3. **Set up DNS, parameters, CodePipeline, and approval from the client site.** (Estimated Time: 5 hours)
   - Set up the source as an S3 ZIP file.
   - Set up approval steps and SNS notifications.
4. **Use a Lambda function to copy the ZIP file to the required source filename for CodePipeline deployment.** (Estimated Time: 5 hours)
![image](https://github.com/user-attachments/assets/90d4dc33-dece-4698-8562-5be1ca135c69)

## Step-by-Step Instructions

### 1. Output CodePipeline and CloudFormation Artifacts to Separate S3 Buckets

- **Create S3 Buckets:**
  - Create two S3 buckets:
    - One for storing CodePipeline artifacts, such as source code builds and deployment files.
    - Another for storing CloudFormation templates for infrastructure deployments.

- **Configure CodePipeline:**
  - Within your CodePipeline, configure the output artifacts to be stored in the designated S3 buckets.
  - Example configuration in CodePipeline:
    ```yaml
    Artifacts:
      - name: application-artifacts
        location: s3://<codepipeline-artifacts-bucket>
      - name: cloudformation-templates
        location: s3://<cloudformation-templates-bucket>
    ```

### 2. Copy Source Artifacts to Client Buckets Using a Prefix via Lambda (5 Hours)

- **Create a Lambda Function:**
  - Develop a Lambda function that listens for changes in the main S3 bucket. When triggered, it copies the source artifacts to client-specific buckets using a specified prefix.

- **Configure the Lambda Trigger:**
  - Set up an S3 trigger for the source bucket that invokes the Lambda function on object creation events.

- **Sample Lambda Code (Python using `boto3`):**
  ```python
  import boto3

  s3 = boto3.client('s3')

  def lambda_handler(event, context):
      source_bucket = '<source-bucket>'
      client_buckets = ['<client-bucket-1>', '<client-bucket-2>']
      prefix = '<prefix>'
      
      for record in event['Records']:
          key = record['s3']['object']['key']
          
          # Copy the object to each client bucket with the specified prefix
          for bucket in client_buckets:
              s3.copy_object(
                  CopySource={'Bucket': source_bucket, 'Key': key},
                  Bucket=bucket,
                  Key=f"{prefix}/{key}"
              )
  ```

- **Testing:**
  - Upload a test file to the source bucket to verify that the Lambda function successfully copies the file to the client buckets with the defined prefix.

### 3. Set Up DNS, Parameters, CodePipeline, and Approval from Client Site (5 Hours)

#### Step 3.1: Set Up DNS and Parameters

- **DNS Configuration:**
  - Use AWS Route 53 (or the client’s DNS provider) to set up necessary DNS records for the application.

- **Parameter Store:**
  - Use AWS Systems Manager Parameter Store to keep environment-specific parameters, such as API keys and database credentials. These parameters will be referenced by CodePipeline and CloudFormation.

#### Step 3.2: Set Up CodePipeline

- **Create CodePipeline:**
  - Define the pipeline with stages including:
    - **Source Stage:** Set the source to an S3 bucket containing a ZIP file.
    - **Approval Stage:** Add a manual approval action.
    - **Deploy Stage:** Deploy using CodeDeploy or CloudFormation templates.

- **Sample CodePipeline Configuration (YAML):**
  ```yaml
  Source:
    S3:
      Bucket: <source-bucket-name>
      ObjectKey: <source.zip>

  Approval:
    Manual:
      NotifyEmail: <notification-email>
  ```

#### Step 3.3: Set Up Approval and SNS Notification

- **Create an SNS Topic:**
  - Create an SNS topic to send notifications for approvals or pipeline status updates.
  - Subscribe necessary team members to the SNS topic for real-time notifications.

- **Approval Stage:**
  - Add an approval step to the pipeline that sends a notification to the SNS topic, prompting manual review before proceeding to deployment.

### 4. Use a Lambda Function to Copy the ZIP File to the Source Filename for CodePipeline Deployment (5 Hours)

- **Create/Modify the Lambda Function:**
  - This Lambda function modifies the uploaded file to match the required source filename (e.g., `source.zip`) for the CodePipeline to pick up for deployment.

- **Sample Lambda Code (Python):**
  ```python
  import boto3

  s3 = boto3.client('s3')

  def lambda_handler(event, context):
      source_bucket = '<source-bucket>'
      file_key = event['Records'][0]['s3']['object']['key']
      target_key = 'source.zip'
      
      # Copy the file to the desired name for CodePipeline
      s3.copy_object(
          CopySource={'Bucket': source_bucket, 'Key': file_key},
          Bucket=source_bucket,
          Key=target_key
      )
  ```

- **Configure Trigger:**
  - Configure the Lambda function to trigger on object creation events in the source S3 bucket.

- **Testing:**
  - Upload a file to the source S3 bucket and verify if the Lambda function renames it to `source.zip` for CodePipeline deployment.

## Notes

- **IAM Permissions:** Ensure that all necessary IAM roles and policies are in place for S3, Lambda, CodePipeline, and other resources to interact securely and correctly.
- **Testing:** Validate each component individually to ensure synchronization, deployment, and notification processes are working correctly.
- **Time Estimates:** The time estimates are based on average setup durations and may vary depending on specific client requirements and configurations.
