# 📘 OmniData: Serverless & Container Hybrid Architecture Guide

## Phase 1: The Foundation (Database & Ingest)
**Goal:** Create the storage layer and a way to write data to it.

### 1. Create DynamoDB Table
* **Name:** `OmniData`
* **Partition Key:** `user_id` (String)
* **Settings:** Default (Provisioned or On-Demand)

### 2. Create "Ingest" Lambda Function
* **Function Name:** `OmniIngest`
* **Runtime:** Python 3.12
* **Code (`lambda_function.py`):**

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('OmniData')

def lambda_handler(event, context):
    try:
        # Parse body (handles both direct invoke and API Gateway proxy)
        body = json.loads(event['body']) if 'body' in event else event
        
        table.put_item(Item=body)
        
        return {
            'statusCode': 200,
            'headers': {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Methods': 'POST,OPTIONS'
            },
            'body': json.dumps('User added successfully!')
        }
    except Exception as e:
        return {'statusCode': 500, 'body': str(e)}
```
### 3. Configure API Gateway (Write Path)
* **Create REST API:** OmniAPI
* **Create Resource:** /stream
* **Create Method:** POST -> Integration Type: Lambda Function (OmniIngest)
* **Action:** Enable CORS on the resource.

## Phase 2: The Retrieval (Read Path)
**Goal:** Create a function to fetch data and expose it via API.

### 1. Create "Getter" Lambda Function
**Function Name:** ServerlessUserGetter
**Runtime:** Python 3.12
* **Code (`lambda_function.py`):**
```Python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('OmniData')

def lambda_handler(event, context):
    user_id = event['queryStringParameters']['user_id']
    response = table.get_item(Key={'user_id': user_id})
    
    # (Phase 6 Code will go here later)

    if 'Item' in response:
        return {
            'statusCode': 200,
            'headers': { 'Access-Control-Allow-Origin': '*' },
            'body': json.dumps(response['Item'])
        }
    else:
        return {
            'statusCode': 404,
            'headers': { 'Access-Control-Allow-Origin': '*' },
            'body': json.dumps({'message': 'User not found'})
        }
```
### 2. Configure API Gateway (Read Path)
* **Create Resource:** /get-user
* **Create Method:** GET -> Integration Type: Lambda (ServerlessUserGetter)
* **Action:** Enable CORS.
* **Deploy API:** Create a Stage named Dev.

## Phase 3: Security & Observability
**Goal:** Secure the API with keys and monitor traffic.

### 1. API Keys & Usage Plans
* **API Keys:** Create new key OmniKey. Copy the value.
* **Usage Plan:** Create BasicPlan (Throttle: 10 req/sec, Quota: 500 req/day).
* **Link:** Add OmniKey to BasicPlan and associate with OmniAPI (Dev Stage).
* **Enforce:** On API Gateway -> /get-user -> GET method -> Set API Key Required: true.

### 2. Monitoring (X-Ray & CloudWatch)
* **Lambda:** Configuration -> Monitoring -> Toggle "Active Tracing" (X-Ray).
* **API Gateway:** Stage -> Logs/Tracing -> Enable X-Ray Tracing.
* **CloudWatch Alarm:** Create Alarm on OmniAPI -> Metric: Count -> Condition: > 5 for 1 minute.
* **Action:** Send notification to SNS Topic HighTrafficAlert (Email endpoint).

## Phase 4: Frontend Hosting
**Goal:** Host the static website.

### 1. S3 Bucket
* **Create Bucket:** omni-data-frontend-[yourname]
* **Upload:** index.html (Use the HTML code provided in the project files).
* **Note:** We do not turn on Static Website Hosting yet because we are using CloudFront.

## Phase 5: Professional Security (Cognito & CloudFront)
**Goal:** Add User Login and HTTPS.

### 1. CloudFront Distribution
* **Origin Domain:** Select your S3 bucket.
* **Origin Access Control (OAC):** Create new (Restricts direct S3 access).
* **Viewer Protocol Policy:** Redirect HTTP to HTTPS.
* **Default Root Object:** index.html.

### 2. Amazon Cognito
* **Create User Pool:** OmniUsers.
* **Sign-in options:** Email.
* **App Client:** OmniWebClient.
* **Callback URL:** Paste your CloudFront URL (e.g., https://d123.cloudfront.net).

### 3. API Gateway Authorizer
Go to OmniAPI -> Authorizers -> Create New.
* **Type:** Cognito.
* **Token Source:** Authorization.
* **Attach:** Go to /get-user -> GET Method -> Method Request -> Set Authorization to OmniCognito.

## Phase 6: The Asynchronous Architecture
**Goal:** Decouple the logging process using a Queue.

### 1. SQS Queue
* **Create Queue:** OmniQueue (Standard).
* **Copy URL:** https://sqs.us-east-1.amazonaws.com/.../OmniQueue

### 2. Update "Getter" Lambda
* **Add Permission:** Attach AmazonSQSFullAccess to the Lambda IAM Role.

* **Code (`lambda_serverlessgetter.py`):**
```Python
# Add inside lambda_handler
sqs = boto3.client('sqs')
sqs.send_message(
    QueueUrl='YOUR_SQS_URL',
    MessageBody=json.dumps({'user': user_id, 'action': 'search'})
)
```

## Phase 7: The Heavy Lifter (ECS Fargate)
**Goal:** Process the Queue using a Container.

### 1. Preparation (S3 & ECR)
* **Create Analytics Bucket:** omni-analytics-data-[yourname].
* **Create ECR Repository:** omni-worker-repo.

### 2. The Worker Code
**Create worker.py (The logic that polls SQS and writes to S3).**
```
import boto3
import json
import os
import time

# Get configurations from Environment Variables
# We will set the ACTUAL values later in the ECS Console
SQS_QUEUE_URL = os.environ.get('SQS_QUEUE_URL') 
S3_BUCKET_NAME = os.environ.get('S3_BUCKET_NAME')

# Initialize Clients
sqs = boto3.client('sqs', region_name='us-east-1')
s3 = boto3.client('s3', region_name='us-east-1')

def process_messages():
    print(f"Worker started. Listening to {SQS_QUEUE_URL}...")
    
    # Validation check
    if not SQS_QUEUE_URL or not S3_BUCKET_NAME:
        print("ERROR: Environment variables SQS_QUEUE_URL or S3_BUCKET_NAME are missing.")
        return

    while True:
        try:
            # Poll SQS for messages (Long Polling: 20 seconds)
            response = sqs.receive_message(
                QueueUrl=SQS_QUEUE_URL,
                MaxNumberOfMessages=1,
                WaitTimeSeconds=20
            )

            if 'Messages' in response:
                for message in response['Messages']:
                    print("Message received!")
                    body = message['Body']
                    receipt_handle = message['ReceiptHandle']

                    # Process Data (Save to S3)
                    file_name = f"search_log_{message['MessageId']}.json"
                    
                    s3.put_object(
                        Bucket=S3_BUCKET_NAME,
                        Key=file_name,
                        Body=body
                    )
                    print(f"Saved to S3: {file_name}")

                    # Delete from Queue
                    sqs.delete_message(
                        QueueUrl=SQS_QUEUE_URL,
                        ReceiptHandle=receipt_handle
                    )
                    print("Message deleted from queue.")
            else:
                print("No messages. Waiting...")
                
        except Exception as e:
            print(f"Error: {str(e)}")
            time.sleep(5)

if __name__ == "__main__":
    process_messages()
```

**Create Dockerfile:**

```Dockerfile

FROM python:3.9-slim
WORKDIR /app
COPY . .
RUN pip install boto3
CMD ["python", "worker.py"]
```
**Create requirements.txt**
```
boto3
```
### 3. Build & Push (CloudShell)
```
Bash

# Login
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin [YOUR_ACCOUNT_ID].dkr.ecr.us-east-1.amazonaws.com

# Build
docker build -t omni-worker-repo .

# Tag & Push
docker tag omni-worker-repo:latest [YOUR_ACCOUNT_ID][.dkr.ecr.us-east-1.amazonaws.com/omni-worker-repo:latest](https://.dkr.ecr.us-east-1.amazonaws.com/omni-worker-repo:latest)
docker push [YOUR_ACCOUNT_ID][.dkr.ecr.us-east-1.amazonaws.com/omni-worker-repo:latest](https://.dkr.ecr.us-east-1.amazonaws.com/omni-worker-repo:latest)
```

### 4. Deploy to ECS Fargate

* **Cluster:** Create Cluster OmniCluster (Fargate).
* **Task Role:** Create OmniWorkerTaskRole (Permissions: AmazonSQSFullAccess, AmazonS3FullAccess).
* **Task Definition:** That has two key and value SQS url and S3 Bucket and attached the IAM policy in task execution and task role
* **Image:** URI from ECR.
* **Environment Variables:** SQS_URL, BUCKET_NAME.
* **Run Task:**  
Launch Type: Fargate.
Network: Select VPC -> Auto-assign Public IP: ENABLED (Critical for reaching SQS).
