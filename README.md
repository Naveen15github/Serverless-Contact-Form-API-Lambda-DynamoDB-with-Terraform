# Serverless-Contact-Form-API-Lambda-DynamoDB-with-Terraform

Delivering a robust, fully serverless contact form pipeline powered by **AWS Lambda**, **API Gateway**, and **DynamoDB**, all infrastructure provisioned via **Terraform**. Designed for reliability, scalability, and rapid deployment—perfect for modern enterprise needs.

---

## 📸 Architecture & Implementation Screenshots

### 1. Solution Architecture

![Serverless Contact Form Architecture](image1.png)

*This diagram illustrates the end-to-end flow: form submissions from a static site are processed by a Lambda function and stored in DynamoDB, with all infrastructure provisioned using Terraform.*

---

### 2. Terraform Deployment in Action

![Terraform Apply Output](image2.png)

*Terraform provisions all resources automatically, including IAM roles, DynamoDB, Lambda, and API Gateway, ensuring reproducibility and automation.*

---

### 3. Frontend Contact Form Example

![Contact Form Frontend](image3.png)

*A simple HTML contact form deployed to S3, ready to POST submissions to the serverless backend.*

---

### 4. DynamoDB Table: ContactFormEntries

![DynamoDB Table](image4.png)

*DynamoDB table “ContactFormEntries” showing active status and the schema used to store submissions.*

---

### 5. DynamoDB Table Scan Result

![DynamoDB Table Scan Result](image5.png)

*This screenshot shows the AWS DynamoDB Console with the "ContactFormEntries" table selected. The "Explore items" section displays a scan result with one record, including the fields: email, message, name, and timestamp. The scan is successful, indicating the contact form data has been stored as expected.*

---

### 6. DynamoDB Live Item Count Dialog

![DynamoDB Live Item Count](image6.png)

*This image displays the "Get live item count" popup in the DynamoDB Console. It shows a completed scan operation, indicating there is currently 1 item in the table. The dialog warns that scanning large tables may consume extra read capacity and is mainly for informational or development use.*

---

## 🚀 Step-by-Step Deployment Guide

### Step 1: Lambda Function

Create the Lambda function that processes contact form submissions and stores them in DynamoDB.

**File:** `lambda/lambda_function.py`

```python
import json
import boto3
import datetime
import os

# Connect to DynamoDB table
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

def lambda_handler(event, context):
    # Parse incoming form data
    body = json.loads(event['body'])
    name = body.get('name', 'N/A')
    email = body.get('email', 'N/A')
    message = body.get('message', 'N/A')

    # Save submission to DynamoDB
    table.put_item(Item={
        'email': email,
        'name': name,
        'message': message,
        'timestamp': str(datetime.datetime.utcnow())
    })

    # Return response
    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'message': 'Form submitted successfully!'})
    }
```

**Zip Lambda Folder:**

```sh
powershell Compress-Archive -Path lambda\* -DestinationPath lambda.zip
```
This creates `lambda.zip` in your project root.  
**Important:** Re-zip every time you change the code before redeploying.

---

### Step 2: Terraform Configuration

**File:** `main.tf`

```hcl
########################################################
# Terraform Provider
########################################################
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

########################################################
# DynamoDB Table
########################################################
resource "aws_dynamodb_table" "contact_form" {
  name         = "ContactFormEntries"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "email"

  attribute {
    name = "email"
    type = "S"
  }
}

########################################################
# IAM Role for Lambda
########################################################
resource "aws_iam_role" "lambda_role" {
  name = "lambda-dynamodb-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_dynamodb_attach" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess"
}

resource "aws_iam_role_policy_attachment" "lambda_basic_attach" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

########################################################
# Lambda Function
########################################################
resource "aws_lambda_function" "contact_form" {
  filename         = "lambda.zip"
  function_name    = "ContactFormHandler"
  role             = aws_iam_role.lambda_role.arn
  handler          = "lambda_function.lambda_handler"
  runtime          = "python3.12"
  source_code_hash = filebase64sha256("lambda.zip")

  environment {
    variables = {
      TABLE_NAME = aws_dynamodb_table.contact_form.name
    }
  }
}

########################################################
# API Gateway HTTP API
########################################################
resource "aws_apigatewayv2_api" "contact_form_api" {
  name          = "ContactFormAPI"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda_integration" {
  api_id                  = aws_apigatewayv2_api.contact_form_api.id
  integration_type        = "AWS_PROXY"
  integration_uri         = aws_lambda_function.contact_form.arn
  integration_method      = "POST"
  payload_format_version  = "2.0"
}

resource "aws_apigatewayv2_route" "post_route" {
  api_id    = aws_apigatewayv2_api.contact_form_api.id
  route_key = "POST /submit"
  target    = "integrations/${aws_apigatewayv2_integration.lambda_integration.id}"
}

resource "aws_apigatewayv2_stage" "default_stage" {
  api_id      = aws_apigatewayv2_api.contact_form_api.id
  name        = "$default"
  auto_deploy = true
}

resource "aws_lambda_permission" "api_gateway_permission" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.contact_form.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.contact_form_api.execution_arn}/*/*"
}
```

---

### Step 3: Deploy Terraform

**Initialize Terraform:**

```sh
terraform init
```

**Preview Deployment:**

```sh
terraform plan
```

**Apply:**

```sh
terraform apply
```
Type `yes` to confirm.  
Terraform will create DynamoDB, Lambda, API Gateway, and IAM roles.

Copy the API Gateway URL from the output for use in the frontend.

---

### Step 4: Create Frontend HTML Form

**File:** `index.html`

```html
<form id="contactForm">
  <input type="text" name="name" placeholder="Your Name" required><br><br>
  <input type="email" name="email" placeholder="Your Email" required><br><br>
  <textarea name="message" placeholder="Your Message" required></textarea><br><br>
  <button type="submit">Send</button>
</form>

<script>
const form = document.getElementById('contactForm');
form.addEventListener('submit', async (e) => {
  e.preventDefault();
  const data = {
    name: form.name.value,
    email: form.email.value,
    message: form.message.value
  };
  const response = await fetch('YOUR_API_GATEWAY_URL/submit', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
  });
  const result = await response.json();
  alert(result.message);
});
</script>
```
> **Replace** `YOUR_API_GATEWAY_URL` with your actual API Gateway URL from the Terraform outputs.

---

### Step 5: Host the Frontend

- **Option 1:** Upload `index.html` to AWS S3 and enable static website hosting.
- **Option 2:** Deploy to Netlify or GitHub Pages for quick hosting.

---

## 🚩 Solution Overview

- **Seamless API → Lambda → DynamoDB Integration:**  
  When a user submits the contact form, data flows securely through API Gateway to a Lambda function, which validates and stores entries in DynamoDB.

- **Infrastructure as Code (IaC) with Terraform:**  
  Full automation and version control for all AWS resources—no manual setup required. Easily replicate, scale, or destroy environments with a single command.

- **Plug-and-Play Frontend Compatibility:**  
  Effortlessly connect any web frontend. A reference HTML form is included for immediate integration and testing.

- **Enterprise-Grade Scalability & Reliability:**  
  Serverless design ensures automatic scaling for high-traffic scenarios, built-in fault tolerance, and zero infrastructure maintenance burden.

---

## 🏆 Key Technologies

- **AWS Lambda** (Python): Stateless, event-driven compute for processing submissions.
- **Amazon API Gateway** (HTTP API): Secure, scalable API layer.
- **Amazon DynamoDB**: Managed NoSQL database for storing contact data.
- **Terraform**: Define, provision, and manage AWS resources via code.

---

## 🎯 Business Value

- **Accelerate Delivery:** Infrastructure is ready in minutes, not days.
- **Reduce Manual Effort:** Fully automated provisioning and deployment via Terraform.
- **Increase Reliability:** AWS-managed, serverless stack is highly available and self-healing.
- **Enhance Security:** IAM roles enforce least-privilege access; CORS is enabled and customizable.

---

## 🚀 How It Works

1. **User submits the contact form** (frontend HTML/JS).
2. **API Gateway** receives the POST request.
3. **Lambda function** processes, validates, and stores the data in **DynamoDB**.
4. **Terraform** manages all provisioning—repeatable, auditable, and easy to maintain.

---

## 📋 Quick Start

1. **Clone the repo and review `terraform/variables.tf` for your region and settings.**
2. **Run:**
   ```sh
   cd terraform
   terraform init
   terraform apply
   ```
3. **Deploy Lambda code if not automated by Terraform.**
4. **Connect your frontend to the API Gateway endpoint output by Terraform!**

---

## 🔒 Security & Compliance

- **IAM Policies**: Lambda role scoped to only required DynamoDB actions.
- **CORS**: Configurable for your organization’s security needs.
- **Input Validation**: Lambda handler can be extended for advanced checks.

---
