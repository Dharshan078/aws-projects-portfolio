Phase 1 (The Foundation): Created DynamoDB and an "Ingest" Lambda to write data.

Phase 2 (The API): Built a "Getter" Lambda and connected both to API Gateway.

Phase 3 (Security & Monitoring): Added API Keys and Usage Plans to throttle traffic. Enabled X-Ray for tracing and configured CloudWatch Alarms + SNS to email alerts on high traffic.

Phase 4 (The Frontend): Hosted a static website on S3.

Phase 5 (Professional Security): Moved to CloudFront (HTTPS), implemented Amazon Cognito (User Login), and secured the API with OAuth Tokens.

Phase 6 (The Decoupling): Connected the Getter Lambda to SQS to asynchronously log searches.

Phase 7 (The Heavy Lifter): Built a Docker container, pushed to ECR, and ran it on ECS Fargate to process the queue and save logs to S3.

Phase 8 (The UI Update): Updated the webpage to support POST requests (adding users) and handled CORS issues.
