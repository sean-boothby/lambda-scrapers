# Web Scraper to AWS Lambda: Complete Setup Guide

This guide walks you through converting any Python web scraper into an AWS Lambda function using the Serverless framework. Based on real-world experience with Old Navy category scraping, this guide covers everything from initial setup to deployment.

## Prerequisites

- Python 3.8+ installed
- AWS account with access credentials
- Basic familiarity with Python and web scraping concepts

## Step 1: Initial Setup

### 1.1 Create Project Directory
```bash
mkdir my-scraper-project
cd my-scraper-project
```

### 1.2 Set Up Python Virtual Environment
```bash
# Create virtual environment
python -m venv venv

# Activate virtual environment
# On Windows:
venv\Scripts\activate
# On macOS/Linux:
source venv/bin/activate
```

### 1.3 Install Python Dependencies
```bash
# Install core scraping libraries
pip install requests beautifulsoup4

# Install AWS and Serverless dependencies
pip install boto3

# Create requirements.txt
pip freeze > requirements.txt
```

## Step 2: Install Node.js and Serverless Framework

### 2.1 Install Node.js
Download and install Node.js from [nodejs.org](https://nodejs.org/) (LTS version recommended)

### 2.2 Install Serverless Framework
```bash
# Install Serverless Framework globally
npm install -g serverless

# Verify installation
serverless --version
```

### 2.3 Initialize Serverless Project
```bash
# Initialize new Serverless project
serverless create --template aws-python3 --name my-scraper

# This creates serverless.yml and handler.py files
```

## Step 3: Configure AWS Credentials

### 3.1 Install AWS CLI
Download and install AWS CLI from [aws.amazon.com/cli](https://aws.amazon.com/cli/)

### 3.2 Configure AWS Credentials
```bash
# Configure AWS credentials
aws configure

# You'll be prompted for:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region (e.g., us-east-1)
# - Default output format (json)
```

## Step 4: Project Structure

Your project should look like this:
```
my-scraper-project/
├── venv/
├── requirements.txt
├── serverless.yml
├── handler.py
├── scraper.py
└── package.json
```

## Step 5: Create Your Scraper

### 5.1 Create scraper.py
Create your main scraping logic in `scraper.py`:

```python
import requests
from bs4 import BeautifulSoup
import json
import logging
from datetime import datetime
import uuid

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def scrape_website():
    """Your main scraping function"""
    url = "https://example.com"
    
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
    }
    
    try:
        response = requests.get(url, headers=headers, timeout=30)
        response.raise_for_status()
        
        soup = BeautifulSoup(response.text, "html.parser")
        
        # Your scraping logic here
        data = []
        # Example: extract links
        links = soup.find_all('a')
        for link in links:
            data.append({
                "text": link.get_text(strip=True),
                "href": link.get("href", "")
            })
        
        return {
            "success": True,
            "data": data,
            "count": len(data)
        }
        
    except Exception as e:
        logger.error(f"Scraping failed: {str(e)}")
        return {
            "success": False,
            "error": str(e)
        }
```

### 5.2 Create handler.py
Create the Lambda handler function:

```python
import json
import logging
from scraper import scrape_website

# Set up logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """AWS Lambda handler function"""
    try:
        # Call your scraper
        result = scrape_website()
        
        # Return response with CORS headers
        return {
            "statusCode": 200,
            "headers": {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*",
                "Access-Control-Allow-Headers": "Content-Type",
                "Access-Control-Allow-Methods": "GET, POST, OPTIONS"
            },
            "body": json.dumps(result)
        }
        
    except Exception as e:
        logger.error(f"Lambda execution failed: {str(e)}")
        return {
            "statusCode": 500,
            "headers": {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*"
            },
            "body": json.dumps({
                "error": "Internal server error",
                "message": str(e)
            })
        }
```

## Step 6: Configure Serverless

### 6.1 Update serverless.yml
```yaml
service: my-scraper

frameworkVersion: '3'

provider:
  name: aws
  runtime: python3.9
  region: us-east-1
  timeout: 60
  memorySize: 512
  
  # Environment variables (if needed)
  environment:
    VARIABLE_NAME: value

plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: non-linux
    layer:
      name: python-deps
      description: Python dependencies for my-scraper
    noDeploy:
      - coverage
      - pytest

functions:
  scrape:
    handler: handler.lambda_handler
    events:
      - http:
          path: scrape
          method: get
          cors: true
```

### 6.2 Install Serverless Python Requirements Plugin
```bash
# Install the plugin
npm install --save-dev serverless-python-requirements

# This plugin handles Python dependencies for Lambda
```

## Step 7: Handle Dependencies

### 7.1 Update requirements.txt
Make sure your `requirements.txt` includes all necessary packages:

```
requests==2.31.0
beautifulsoup4==4.12.2
boto3==1.34.0
```

### 7.2 Install Plugin Dependencies
```bash
# Install serverless-python-requirements
npm install --save-dev serverless-python-requirements
```

## Step 8: Testing Locally

### 8.1 Test Your Scraper
```bash
# Test your scraper locally
python -c "from scraper import scrape_website; print(scrape_website())"
```

### 8.2 Test Lambda Handler Locally
```bash
# Test the handler function
python -c "from handler import lambda_handler; print(lambda_handler({}, {}))"
```

## Step 9: Deployment

### 9.1 Deploy to AWS
```bash
# Deploy your function
serverless deploy

# This will:
# - Package your Python code
# - Create a Lambda layer with dependencies
# - Deploy to AWS Lambda
# - Create API Gateway endpoint
```

### 9.2 Verify Deployment
```bash
# List your functions
serverless info

# Test the deployed function
serverless invoke -f scrape
```

### 9.3 Alternative Invocation Methods

#### Direct Lambda Invocation (No API Gateway Timeout)
```bash
# Synchronous invocation (waits for result, up to 15 minutes)
serverless invoke -f scrape

# Asynchronous invocation (returns immediately, runs in background)
serverless invoke -f scrape --async

# With AWS CLI
aws lambda invoke --function-name my-scraper-dev-scrape response.json
aws lambda invoke --function-name my-scraper-dev-scrape --invocation-type Event response.json
```

#### Scheduled Execution
Add to your `serverless.yml`:
```yaml
functions:
  scrape:
    handler: handler.lambda_handler
    events:
      - schedule: rate(1 hour)  # Run every hour
      - schedule: cron(0 12 * * ? *)  # Run daily at 12 PM UTC
      - http:
          path: scrape
          method: get
          cors: true
```

#### Manual Trigger via AWS Console
1. Go to AWS Lambda Console
2. Select your function
3. Click "Test" tab
4. Create test event and run

#### Programmatic Invocation
```python
import boto3

lambda_client = boto3.client('lambda')

# Synchronous
response = lambda_client.invoke(
    FunctionName='my-scraper-dev-scrape',
    InvocationType='RequestResponse'
)

# Asynchronous
response = lambda_client.invoke(
    FunctionName='my-scraper-dev-scrape',
    InvocationType='Event'
)
```

## Step 10: Advanced Features

### 10.1 Adding Proxy Support
If your scraper needs to use proxies:

```python
# In scraper.py
import os

def scrape_with_proxy():
    proxy_username = os.environ.get('PROXY_USERNAME')
    proxy_password = os.environ.get('PROXY_PASSWORD')
    proxy_host = os.environ.get('PROXY_HOST')
    
    proxies = {
        "http": f"http://{proxy_username}:{proxy_password}@{proxy_host}",
        "https": f"http://{proxy_username}:{proxy_password}@{proxy_host}"
    }
    
    response = requests.get(url, headers=headers, proxies=proxies, timeout=30)
```

Add to `serverless.yml`:
```yaml
provider:
  environment:
    PROXY_USERNAME: your_username
    PROXY_PASSWORD: your_password
    PROXY_HOST: proxy.example.com:8080
```

### 10.2 Adding Retry Logic
```python
import time

def scrape_with_retry(max_retries=3, delay=3):
    for attempt in range(max_retries):
        try:
            result = scrape_website()
            if result.get('success'):
                return result
        except Exception as e:
            if attempt < max_retries - 1:
                logger.warning(f"Attempt {attempt + 1} failed, retrying...")
                time.sleep(delay)
                continue
            else:
                raise
```

### 10.3 Handling Compressed Responses
```python
import gzip
import zlib

def handle_compressed_response(response):
    content_encoding = response.headers.get('content-encoding', '').lower()
    
    if content_encoding == 'gzip':
        content = gzip.decompress(response.content).decode('utf-8')
    elif content_encoding == 'deflate':
        content = zlib.decompress(response.content).decode('utf-8')
    else:
        content = response.text
    
    return content
```

## Step 11: Monitoring and Debugging

### 11.1 View Logs
```bash
# View CloudWatch logs
serverless logs -f scrape -t

# Or use AWS CLI
aws logs tail /aws/lambda/my-scraper-dev-scrape --follow
```

### 11.2 Common Issues and Solutions

#### Issue: Module not found
**Solution**: Ensure all dependencies are in `requirements.txt` and the serverless-python-requirements plugin is installed.

#### Issue: Timeout errors
**Solution**: Increase timeout in `serverless.yml`:
```yaml
provider:
  timeout: 60  # seconds
```

#### Issue: Memory errors
**Solution**: Increase memory allocation:
```yaml
provider:
  memorySize: 1024  # MB
```

#### Issue: API Gateway timeout
**Solution**: API Gateway has a 29-second timeout. For longer operations, consider:

**Option 1: Direct Lambda Invocation (Recommended for long-running tasks)**
```bash
# Invoke Lambda directly (bypasses API Gateway timeout)
serverless invoke -f scrape

# Or with AWS CLI
aws lambda invoke --function-name my-scraper-dev-scrape response.json
```

**Option 2: Asynchronous Invocation**
```bash
# Invoke asynchronously (returns immediately, runs in background)
serverless invoke -f scrape --async

# Or with AWS CLI
aws lambda invoke --function-name my-scraper-dev-scrape --invocation-type Event response.json
```

**Option 3: Scheduled Execution**
Add to `serverless.yml`:
```yaml
functions:
  scrape:
    handler: handler.lambda_handler
    events:
      - schedule: rate(1 hour)  # Run every hour
      - http:
          path: scrape
          method: get
          cors: true
```

**Option 4: EventBridge Trigger**
```yaml
functions:
  scrape:
    handler: handler.lambda_handler
    events:
      - eventBridge:
          schedule: rate(1 hour)
      - http:
          path: scrape
          method: get
          cors: true
```

## Step 12: Production Considerations

### 12.1 Environment Variables
Use different configurations for different stages:

```yaml
provider:
  environment:
    STAGE: ${self:provider.stage}
    LOG_LEVEL: ${self:custom.logLevel.${self:provider.stage}}

custom:
  logLevel:
    dev: DEBUG
    prod: INFO
```

### 12.2 Security
- Never commit AWS credentials to version control
- Use IAM roles with minimal required permissions
- Consider using AWS Secrets Manager for sensitive data

### 12.3 Cost Optimization
- Monitor Lambda execution times and memory usage
- Consider using Lambda Provisioned Concurrency for consistent performance
- Set up CloudWatch alarms for monitoring

## Step 13: Cleanup

### 13.1 Remove Deployment
```bash
# Remove all deployed resources
serverless remove
```

### 13.2 Clean Local Environment
```bash
# Deactivate virtual environment
deactivate

# Remove virtual environment
rm -rf venv

# Remove node_modules
rm -rf node_modules
```

## Troubleshooting

### Common Error: "Unable to import module"
- Check that all dependencies are in `requirements.txt`
- Ensure serverless-python-requirements plugin is installed
- Verify Python runtime version matches your local environment

### Common Error: "Request timeout"
- Increase Lambda timeout in `serverless.yml`
- Optimize your scraper for faster execution
- Consider using async processing for long-running tasks

### Common Error: "Memory exceeded"
- Increase memory allocation in `serverless.yml`
- Optimize your code to use less memory
- Consider processing data in smaller chunks

## Next Steps

Once your basic scraper is working:

1. **Add error handling**: Implement robust error handling and retry logic
2. **Add monitoring**: Set up CloudWatch alarms and dashboards
3. **Optimize performance**: Profile your code and optimize bottlenecks
4. **Add scheduling**: Use EventBridge to run your scraper on a schedule
5. **Add data storage**: Store results in DynamoDB, S3, or other AWS services
6. **Add notifications**: Send results via SNS or SES

## Resources

- [Serverless Framework Documentation](https://www.serverless.com/framework/docs/)
- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [Python Requests Documentation](https://requests.readthedocs.io/)
- [BeautifulSoup Documentation](https://www.crummy.com/software/BeautifulSoup/)

---

This guide provides a complete foundation for deploying web scrapers to AWS Lambda. Adapt the examples to your specific scraping needs and requirements. 
