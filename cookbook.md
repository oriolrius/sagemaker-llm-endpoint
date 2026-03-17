# Cookbook: Deploy a Language Model on AWS with SageMaker

This guide walks through deploying a HuggingFace language model on AWS SageMaker with a public OpenAI-compatible API and a web-based chat interface (OpenWebUI). Every step includes verification checkpoints so you know the process is on track before moving forward.

**What you will build:**

```
Browser --> OpenWebUI (EC2) --> API Gateway --> Lambda --> SageMaker vLLM Endpoint (GPU)
```

**Time required:** ~25-30 minutes (most of it is waiting for SageMaker)

**Cost:** ~$0.76/hour while running. You must delete all resources when done.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Configure AWS Credentials](#2-configure-aws-credentials)
3. [Check GPU Quota](#3-check-gpu-quota)
4. [Find Your VPC and Subnet](#4-find-your-vpc-and-subnet)
5. [Run the Lambda Tests](#5-run-the-lambda-tests)
6. [Deploy the Full Stack](#6-deploy-the-full-stack)
7. [Monitor the Deployment](#7-monitor-the-deployment)
8. [Test the API](#8-test-the-api)
9. [Use the Web Chat Interface](#9-use-the-web-chat-interface)
10. [Cleanup (Required)](#10-cleanup-required)
11. [Verify Cleanup](#11-verify-cleanup)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Prerequisites

Before starting, make sure these tools are installed and working.

### Required Software

| Tool | Purpose | Install |
|------|---------|---------|
| **AWS CLI v2** | Interact with AWS services | [Install guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |
| **Git** | Clone the repository | `sudo apt install git` or [git-scm.com](https://git-scm.com/) |
| **uv** | Python package manager | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| **curl** | Test API endpoints | Pre-installed on most systems |
| **zip** | Package Lambda function | `sudo apt install zip` |

### Verify Each Tool

Run all of these and confirm they produce output (not "command not found"):

```bash
aws --version
git --version
uv --version
curl --version
zip --version
```

### Checkpoint

All five commands return version numbers. If any command fails, install that tool before continuing.

---

## 2. Configure AWS Credentials

AWS CLI needs valid credentials to create resources. These credentials come from your AWS account (typically via AWS SSO or an Innovation Sandbox portal).

### Option A: Configure Credentials Manually

If you have an AWS Access Key ID, Secret Access Key, and Session Token:

```bash
aws configure set aws_access_key_id <YOUR_ACCESS_KEY_ID>
aws configure set aws_secret_access_key <YOUR_SECRET_ACCESS_KEY>
aws configure set aws_session_token <YOUR_SESSION_TOKEN>
aws configure set region eu-north-1
```

### Option B: AWS SSO Login

If your organization uses AWS SSO (Identity Center):

```bash
aws configure sso
# Follow the prompts: SSO start URL, region, account, role
```

### Verify Credentials

```bash
aws sts get-caller-identity
```

**Expected output:**

```json
{
    "UserId": "AROA...:your-session-name",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/..."
}
```

### Checkpoint

- The `get-caller-identity` command returns your Account ID (a 12-digit number)
- The region is set to `eu-north-1` (verify with `aws configure get region`)
- Write down your **Account ID** -- you will need it later

### If This Fails

| Error | Cause | Fix |
|-------|-------|-----|
| `Unable to locate credentials` | No credentials configured | Run `aws configure` and enter your keys |
| `ExpiredTokenException` | Session token has expired | Get fresh credentials from your SSO portal |
| `InvalidClientTokenId` | Wrong access key | Double-check the access key ID is correct |

---

## 3. Check GPU Quota

SageMaker requires a GPU instance to run vLLM. Your AWS account must have quota for at least 1 `ml.g4dn.xlarge` instance in `eu-north-1`. New accounts often have a quota of 0 for GPU instances.

### Check via AWS CLI

```bash
aws service-quotas list-service-quotas \
  --service-code sagemaker \
  --region eu-north-1 \
  --query "Quotas[?contains(QuotaName, 'ml.g4dn.xlarge') && contains(QuotaName, 'endpoint')].[QuotaName,Value]" \
  --output table
```

**Expected output:**

```
---------------------------------------------------
| ml.g4dn.xlarge for endpoint usage |  1.0        |
---------------------------------------------------
```

### Check via AWS Console

1. Open the [Service Quotas console for SageMaker in eu-north-1](https://eu-north-1.console.aws.amazon.com/servicequotas/home/services/sagemaker/quotas)
2. In the search box, type `ml.g4dn.xlarge`
3. Find the row **"ml.g4dn.xlarge for endpoint usage"**
4. Check the **Applied quota value** column

### Checkpoint

- The quota value is **1** or higher
- If the quota is **0**, you must request an increase before continuing (see below)

### Request a Quota Increase (if quota is 0)

1. On the Service Quotas page, click the quota name **"ml.g4dn.xlarge for endpoint usage"**
2. Click **"Request quota increase"** (top-right)
3. Enter **1** as the new value
4. Click **"Request"**

Quota increases for GPU instances typically take **a few hours to 3 business days** to be approved. You cannot proceed with the deployment until the quota is at least 1.

---

## 4. Find Your VPC and Subnet

The EC2 instance (which runs the web chat interface) needs a VPC and a **public subnet** to be accessible from the internet.

### What Are VPCs and Subnets?

- **VPC (Virtual Private Cloud)**: An isolated virtual network in your AWS account
- **Subnet**: A range of IP addresses within a VPC. A **public subnet** has a route to the internet via an Internet Gateway
- Every AWS account has a **default VPC** with public subnets -- this is what you should use

### Find VPC via CLI

```bash
aws ec2 describe-vpcs --region eu-north-1 \
  --query 'Vpcs[*].{ID:VpcId,Name:Tags[?Key==`Name`].Value|[0],CIDR:CidrBlock,Default:IsDefault}' \
  --output table
```

**Example output:**

```
-------------------------------------------------------------
|                        DescribeVpcs                       |
+----------------+--------+-----------------------+---------+
|      CIDR      | Default|          ID           |  Name   |
+----------------+--------+-----------------------+---------+
|  172.31.0.0/16 |  True  |  vpc-0abc123def456789 |  None   |
+----------------+--------+-----------------------+---------+
```

Write down the **VPC ID** where `Default` is `True` (e.g., `vpc-0abc123def456789`).

### Find Public Subnet via CLI

Replace `<your-vpc-id>` with the VPC ID from the previous step:

```bash
aws ec2 describe-subnets --region eu-north-1 \
  --filters \
    "Name=vpc-id,Values=<your-vpc-id>" \
    "Name=map-public-ip-on-launch,Values=true" \
  --query 'Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}' \
  --output table
```

**Example output:**

```
------------------------------------------------------
|                    DescribeSubnets                  |
+----------------+-------------------+----------------+
|       AZ       |        ID         |      CIDR      |
+----------------+-------------------+----------------+
|  eu-north-1a   | subnet-0aaa111bbb |  172.31.0.0/20 |
|  eu-north-1b   | subnet-0bbb222ccc |  172.31.16.0/20|
|  eu-north-1c   | subnet-0ccc333ddd |  172.31.32.0/20|
+----------------+-------------------+----------------+
```

Write down **any one** of the Subnet IDs (e.g., `subnet-0aaa111bbb`). Any public subnet works.

### Find VPC and Subnet via AWS Console

1. Open the [VPC Console in eu-north-1](https://eu-north-1.console.aws.amazon.com/vpc/home?region=eu-north-1#vpcs:)
2. Look for the VPC where **Default VPC** column shows **Yes**
3. Copy the **VPC ID**
4. In the left sidebar, click **Subnets** ([direct link](https://eu-north-1.console.aws.amazon.com/vpc/home?region=eu-north-1#subnets:))
5. Click the **gear icon** (top-right of the table) and enable the column **"Auto-assign public IPv4 address"**
6. Find a subnet where that column shows **Yes** -- this is a public subnet
7. Copy the **Subnet ID**

### Checkpoint

You now have two values written down:

| Value | Example | Your Value |
|-------|---------|------------|
| VPC ID | `vpc-0abc123def456789` | _____________ |
| Subnet ID | `subnet-0aaa111bbb` | _____________ |

Both values are required for the next steps.

### If No Public Subnets Appear

This means the VPC has no subnets with internet access. If you are using the **default VPC**, all its subnets should be public. If not:

1. Go to **VPC Console** > **Internet Gateways** -- verify one is attached to your VPC
2. Go to **Route Tables** -- verify there is a route `0.0.0.0/0` pointing to `igw-xxx`
3. Go to **Subnets** > select your subnet > **Actions** > **Edit subnet settings** > enable **"Auto-assign public IPv4 address"**

---

## 5. Run the Lambda Tests

Before deploying to AWS, verify the Lambda proxy code works correctly.

```bash
cd lambda/openai-proxy
uv sync --dev
uv run pytest -v
```

**Expected output:**

```
tests/test_handler.py::TestCreateResponse::test_basic_response PASSED
tests/test_handler.py::TestCreateResponse::test_custom_headers PASSED
...
============================== 19 passed in 0.13s ==============================
```

### Checkpoint

- All 19 tests pass
- No test failures or errors

### If Tests Fail

| Error | Fix |
|-------|-----|
| `uv: command not found` | Install uv: `curl -LsSf https://astral.sh/uv/install.sh \| sh` then restart your terminal |
| `ModuleNotFoundError` | Run `uv sync --dev` again |
| Test assertion errors | Check that you haven't modified `handler.py` -- run `git checkout lambda/openai-proxy/src/` to reset |

Return to the project root when done:

```bash
cd ../..
```

---

## 6. Deploy the Full Stack

This step packages the Lambda function, uploads it to S3, and deploys all AWS resources via CloudFormation.

### Run the Deploy Script

Replace the placeholders with your actual values from Step 4:

```bash
cd infra/

./deploy-full-stack.sh \
  --vpc-id <your-vpc-id> \
  --subnet-id <your-subnet-id>
```

**Example with real values:**

```bash
./deploy-full-stack.sh \
  --vpc-id vpc-0abc123def456789 \
  --subnet-id subnet-0aaa111bbb
```

The script will:

1. Show a summary of what will be created
2. Ask for confirmation (`Continue? [y/N]`) -- type **y** and press Enter
3. Package the Lambda function (~30 seconds)
4. Create an S3 bucket and upload files (~30 seconds)
5. Deploy the CloudFormation stack (~15-20 minutes)
6. Display the endpoints when complete

### What Gets Created

| Resource | Type | Purpose |
|----------|------|---------|
| SageMaker Model | ML Model | vLLM container configuration |
| SageMaker Endpoint Config | Config | Instance type and variant settings |
| SageMaker Endpoint | **GPU Instance** | Runs the language model (ml.g4dn.xlarge) |
| Lambda Function | Compute | Translates OpenAI API format to SageMaker format |
| API Gateway HTTP API | Public API | Exposes the Lambda function at a public URL |
| EC2 Instance | Virtual Machine | Runs the OpenWebUI chat interface |
| Elastic IP | Static IP | Permanent public IP for the EC2 instance |
| 3 IAM Roles | Security | Least-privilege permissions for each service |
| Security Group | Firewall | Allows HTTP/HTTPS/SSH to EC2 |
| S3 Bucket | Storage | Lambda deployment package |

### Checkpoint

The script finishes with output similar to:

```
============================================
Deployment Complete!
============================================

SageMaker Endpoint: openai-sagemaker-stack-vllm-endpoint
API Gateway:        https://abc123xyz.execute-api.eu-north-1.amazonaws.com
OpenWebUI:          http://13.48.xxx.xxx
EC2 Public IP:      13.48.xxx.xxx
```

Write down these values:

| Value | Your Value |
|-------|------------|
| API Gateway URL | _________________________ |
| OpenWebUI URL | _________________________ |
| SageMaker Endpoint Name | _________________________ |

### If the Script Fails Before CloudFormation

| Error | Cause | Fix |
|-------|-------|-----|
| `ERROR: --vpc-id is required` | Missing argument | Add `--vpc-id` with your VPC ID |
| `ExpiredTokenException` | AWS credentials expired | Refresh credentials (Step 2), then re-run |
| `zip: command not found` | zip not installed | `sudo apt install zip` |
| `uv: command not found` | uv not installed | See Step 1 prerequisites |

---

## 7. Monitor the Deployment

The CloudFormation deployment takes 15-20 minutes. Most of that time is SageMaker provisioning the GPU instance and loading the model. You can monitor progress in real-time.

### Monitor via CLI

Watch CloudFormation events in your terminal:

```bash
watch -n 10 "aws cloudformation describe-stack-events \
  --stack-name openai-sagemaker-stack \
  --region eu-north-1 \
  --query 'StackEvents[0:5].[Timestamp,LogicalResourceId,ResourceStatus]' \
  --output table"
```

Press `Ctrl+C` to stop watching.

### Monitor via AWS Console

1. Open the [CloudFormation console in eu-north-1](https://eu-north-1.console.aws.amazon.com/cloudformation/home?region=eu-north-1#/stacks)
2. Click your stack name (**openai-sagemaker-stack**)
3. Click the **Events** tab
4. The page auto-refreshes. Watch for resources transitioning from `CREATE_IN_PROGRESS` to `CREATE_COMPLETE`

### What to Expect (Timeline)

| Time | What Happens |
|------|-------------|
| 0-1 min | IAM roles created |
| 1-2 min | API Gateway, Security Group created |
| 2-3 min | SageMaker Model and Endpoint Config created |
| 3-5 min | Lambda function created, EC2 instance launched |
| 5-20 min | **SageMaker Endpoint: Creating** (pulls container image, downloads model, loads into GPU) |
| 15-20 min | SageMaker Endpoint transitions to **InService** |
| 20 min | Stack status: **CREATE_COMPLETE** |

### Check SageMaker Endpoint Status Separately

The SageMaker endpoint is the slowest resource. To check its status directly:

```bash
aws sagemaker describe-endpoint \
  --endpoint-name openai-sagemaker-stack-vllm-endpoint \
  --region eu-north-1 \
  --query 'EndpointStatus'
```

| Status | Meaning |
|--------|---------|
| `Creating` | Still provisioning -- wait |
| `InService` | Ready to accept requests |
| `Failed` | Something went wrong -- see [Troubleshooting](#12-troubleshooting) |

You can also check the endpoint in the [SageMaker Console > Inference > Endpoints](https://eu-north-1.console.aws.amazon.com/sagemaker/home?region=eu-north-1#/endpoints).

### Checkpoint

- CloudFormation stack status is **CREATE_COMPLETE**
- SageMaker endpoint status is **InService**

### If the Stack Fails (ROLLBACK_IN_PROGRESS)

Find the root cause by looking for the first `CREATE_FAILED` event:

```bash
aws cloudformation describe-stack-events \
  --stack-name openai-sagemaker-stack \
  --region eu-north-1 \
  --query "StackEvents[?ResourceStatus=='CREATE_FAILED'].[LogicalResourceId,ResourceStatusReason]" \
  --output table
```

Common failures and fixes:

| Failed Resource | Error Message | Fix |
|-----------------|---------------|-----|
| SageMakerEndpoint | `ResourceLimitExceeded` | GPU quota is 0 -- request increase (Step 3) |
| LambdaFunction | `S3 error: Access Denied` | S3 bucket region mismatch -- re-run deploy script |
| EC2Instance | `not supported` | Instance type unavailable in AZ -- the script uses `t3.small` which works in eu-north-1 |
| Any IAM resource | `Requires capabilities` | Missing `--capabilities` flag -- the deploy script includes this automatically |

After fixing the issue, delete the failed stack and redeploy:

```bash
aws cloudformation delete-stack --stack-name openai-sagemaker-stack --region eu-north-1
aws cloudformation wait stack-delete-complete --stack-name openai-sagemaker-stack --region eu-north-1
# Then re-run the deploy script
```

---

## 8. Test the API

Once the stack is deployed and the SageMaker endpoint is `InService`, test the API.

### Test 1: List Available Models

Replace `<api-gateway-url>` with your API Gateway URL from Step 6:

```bash
curl <api-gateway-url>/v1/models
```

**Expected output:**

```json
{
  "object": "list",
  "data": [
    {
      "id": "openai-sagemaker-stack-vllm-endpoint",
      "object": "model",
      "created": 1677610602,
      "owned_by": "sagemaker"
    }
  ]
}
```

### Test 2: Send a Chat Completion Request

```bash
curl -X POST <api-gateway-url>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "The future of artificial intelligence is"}],
    "max_tokens": 50,
    "temperature": 0.7
  }'
```

**Expected output:**

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "model": "openai-sagemaker-stack-vllm-endpoint",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "...generated text continues here..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 7,
    "completion_tokens": 50,
    "total_tokens": 57
  }
}
```

### Understanding the Response

The default model is **distilgpt2**, which is a small **base model** (not instruction-tuned). This means:

| Prompt Style | Works? | Example |
|-------------|--------|---------|
| Text completion | Yes | `"The capital of France is"` --> `"Paris, the city of..."` |
| Q&A format | Partial | `"Q: What is AI?\nA:"` --> may continue the pattern |
| Direct question | No | `"What is AI?"` --> produces random text, not an answer |

This is expected behavior. For conversational responses, you would deploy an instruction-tuned model like `meta-llama/Llama-2-7b-chat-hf` (requires a larger GPU instance and more quota).

### Checkpoint

- `/v1/models` returns a JSON list with your endpoint name
- `/v1/chat/completions` returns generated text in the `choices[0].message.content` field
- Both requests complete within ~30 seconds

### If the API Returns Errors

| HTTP Status | Error | Fix |
|-------------|-------|-----|
| 500 | `"SageMaker error"` | Endpoint may not be ready -- check status: `aws sagemaker describe-endpoint --endpoint-name openai-sagemaker-stack-vllm-endpoint --query EndpointStatus` |
| 504 | Gateway Timeout | Request took too long -- try again (first request after deployment can be slow) |
| 403 | Forbidden | API Gateway URL is wrong -- check the CloudFormation outputs |
| Connection refused | Nothing listening | Verify the API Gateway was created: `aws apigatewayv2 get-apis --region eu-north-1` |

---

## 9. Use the Web Chat Interface

OpenWebUI provides a web-based chat interface similar to ChatGPT, connected to your SageMaker model via the API Gateway.

### Access OpenWebUI

Open your browser and navigate to the OpenWebUI URL from Step 6:

```
http://<ec2-public-ip>
```

**Important:** Use `http://` (not `https://`). This setup does not include SSL certificates.

### First-Time Setup

1. The OpenWebUI interface loads in your browser
2. Authentication is **disabled** in this setup, so you may go directly to the chat interface
3. If prompted to create an account, enter any email/password -- this is stored locally on the EC2 instance only
4. The model **openai-sagemaker-stack-vllm-endpoint** should appear in the model selector

### Send a Message

1. Select the model from the dropdown (if not already selected)
2. Type a text completion prompt like: `The most important invention in human history is`
3. Press Enter or click Send
4. The response appears after a few seconds

### Checkpoint

- The OpenWebUI page loads in your browser
- You can select the SageMaker model
- Sending a message returns generated text

### If OpenWebUI Does Not Load

The EC2 instance takes 2-3 minutes after the stack completes to finish installing Docker and starting OpenWebUI. Wait a few minutes and try again.

**Check if Docker is running on the EC2 instance:**

Option 1 -- Via AWS Systems Manager (no SSH key needed):

1. Open the [EC2 Console](https://eu-north-1.console.aws.amazon.com/ec2/home?region=eu-north-1#Instances)
2. Select the instance named `openai-sagemaker-stack-openwebui`
3. Click **Connect** (top button)
4. Click the **Session Manager** tab
5. Click **Connect** -- a terminal opens in your browser
6. Run: `sudo docker ps` to check if the OpenWebUI container is running

Option 2 -- Via SSH (if you provided a key pair during deployment):

```bash
ssh -i ~/.ssh/<your-key>.pem ec2-user@<ec2-public-ip>
sudo docker ps
```

**Check the setup log:**

```bash
sudo cat /var/log/cloud-init-output.log
```

This log shows every command that ran during EC2 startup, including any errors from Docker or the setup script.

---

## 10. Cleanup (Required)

**This stack costs ~$0.76/hour (~$18/day)**. The SageMaker GPU instance is the primary cost driver. Always delete all resources when you are done.

### Delete via Script

```bash
cd infra/
./delete-full-stack.sh --stack-name openai-sagemaker-stack --region eu-north-1
```

The script will:
1. Show what will be deleted
2. Ask for confirmation (`Are you sure? [y/N]`) -- type **y** and press Enter
3. Delete the CloudFormation stack (5-10 minutes)
4. Delete the S3 bucket
5. Confirm cleanup is complete

### Delete via AWS Console

1. Open the [CloudFormation console](https://eu-north-1.console.aws.amazon.com/cloudformation/home?region=eu-north-1#/stacks)
2. Select your stack (**openai-sagemaker-stack**)
3. Click **Delete**
4. Confirm the deletion
5. Wait for the stack status to show **DELETE_COMPLETE** (5-10 minutes)

Then delete the S3 bucket manually:

```bash
# Find the bucket name
aws s3 ls | grep openai-sagemaker-stack

# Empty and delete it
aws s3 rb s3://<bucket-name> --force --region eu-north-1
```

### Checkpoint

- The CloudFormation stack is deleted or shows **DELETE_COMPLETE**
- The S3 bucket no longer exists

---

## 11. Verify Cleanup

Confirm that no billable resources remain. Run each of these commands and verify they return empty results.

### CloudFormation Stack

```bash
aws cloudformation describe-stacks --region eu-north-1 \
  --stack-name openai-sagemaker-stack 2>&1
```

**Expected:** `Stack with id openai-sagemaker-stack does not exist`

### SageMaker Endpoints

```bash
aws sagemaker list-endpoints --region eu-north-1 \
  --query 'Endpoints[?contains(EndpointName, `openai-sagemaker-stack`)]'
```

**Expected:** `[]` (empty array)

### Lambda Functions

```bash
aws lambda list-functions --region eu-north-1 \
  --query 'Functions[?contains(FunctionName, `openai-sagemaker-stack`)]'
```

**Expected:** `[]` (empty array)

### EC2 Instances

```bash
aws ec2 describe-instances --region eu-north-1 \
  --filters "Name=tag:Name,Values=*openai-sagemaker-stack*" \
  --query 'Reservations[*].Instances[?State.Name!=`terminated`].[InstanceId,State.Name]' \
  --output table
```

**Expected:** Empty table (or only `terminated` instances)

### API Gateways

```bash
aws apigatewayv2 get-apis --region eu-north-1 \
  --query 'Items[?contains(Name, `openai-sagemaker-stack`)]'
```

**Expected:** `[]` (empty array)

### Checkpoint

All five commands return empty results. **No ongoing charges.**

---

## 12. Troubleshooting

### Quick Diagnostic Reference

| Symptom | First Command to Run |
|---------|---------------------|
| Stack failing to create | `aws cloudformation describe-stack-events --stack-name openai-sagemaker-stack --query "StackEvents[?ResourceStatus=='CREATE_FAILED'].[LogicalResourceId,ResourceStatusReason]" --output table` |
| SageMaker endpoint not working | `aws sagemaker describe-endpoint --endpoint-name openai-sagemaker-stack-vllm-endpoint --query '[EndpointStatus,FailureReason]'` |
| SageMaker container errors | `aws logs tail /aws/sagemaker/Endpoints/openai-sagemaker-stack-vllm-endpoint --follow` |
| Lambda returning 500 errors | `aws logs filter-log-events --log-group-name /aws/lambda/openai-sagemaker-stack-openai-proxy --filter-pattern "ERROR"` |
| EC2 setup script failed | `aws ec2 get-console-output --instance-id <instance-id> --region eu-north-1 --latest --query 'Output' --output text` |
| Any AWS error | `aws sts get-caller-identity` (check credentials first) |

---

### Credential Issues

#### ExpiredTokenException

```
An error occurred (ExpiredTokenException) when calling the ... operation:
The security token included in the request is expired
```

**Cause:** AWS session tokens from Innovation Sandbox or SSO expire after 1-12 hours.

**Fix:** Obtain fresh credentials from your AWS portal and reconfigure:

```bash
aws configure set aws_access_key_id <NEW_KEY>
aws configure set aws_secret_access_key <NEW_SECRET>
aws configure set aws_session_token <NEW_TOKEN>
aws sts get-caller-identity  # verify the new credentials work
```

#### AccessDeniedException

```
User: arn:aws:sts::123456789012:assumed-role/... is not authorized to perform: ...
```

**Cause:** Your IAM role lacks the required permission. The error message tells you exactly which permission is missing.

**Fix:** Use a role with broader permissions (e.g., `AdministratorAccess` for sandbox environments) or ask your administrator to add the specific permission.

#### Region Mismatch

```
Could not resolve endpoint / Endpoint ... not found
```

**Cause:** The AWS CLI is configured for a different region than where your resources exist.

**Fix:** Always specify `--region eu-north-1` in commands, or set:

```bash
export AWS_DEFAULT_REGION=eu-north-1
```

---

### SageMaker Errors

#### ResourceLimitExceeded (Quota)

```
The account-level service limit 'ml.g4dn.xlarge for endpoint usage' is 0 Instances
```

**Cause:** Your account has zero quota for this GPU instance type.

**Fix:** Request a quota increase (Step 3). You cannot deploy until the quota is at least 1.

#### CUDA Out of Memory

Visible in CloudWatch logs (`/aws/sagemaker/Endpoints/<endpoint-name>`):

```
torch.cuda.OutOfMemoryError: CUDA out of memory
```

**Cause:** The model is too large for the GPU memory (16 GB on ml.g4dn.xlarge).

**Fix:** Use a smaller model. The default `distilgpt2` (82M parameters) fits easily. Models larger than ~7B parameters in fp16 will not fit on a T4 GPU.

#### Endpoint Stuck in "Creating"

**Cause:** Model download or container startup is slow. Normal for first deployment.

**Fix:** Wait up to 25 minutes. If still stuck, check CloudWatch logs:

```bash
aws logs tail /aws/sagemaker/Endpoints/openai-sagemaker-stack-vllm-endpoint --follow
```

If no logs appear at all after 10 minutes, the container failed to start. Check the IAM role has ECR permissions.

---

### CloudFormation Errors

#### ROLLBACK_IN_PROGRESS

**Cause:** One resource failed, triggering automatic rollback of all resources.

**Fix:** Find the root cause (the first `CREATE_FAILED` event):

```bash
aws cloudformation describe-stack-events \
  --stack-name openai-sagemaker-stack \
  --query "StackEvents[?ResourceStatus=='CREATE_FAILED'].[LogicalResourceId,ResourceStatusReason]" \
  --output table
```

After the rollback completes (`ROLLBACK_COMPLETE`), fix the underlying issue and deploy again. The failed stack remains in `ROLLBACK_COMPLETE` state -- delete it first:

```bash
aws cloudformation delete-stack --stack-name openai-sagemaker-stack --region eu-north-1
aws cloudformation wait stack-delete-complete --stack-name openai-sagemaker-stack --region eu-north-1
```

Then re-run the deploy script.

#### Stack Deletion Fails

```
DELETE_FAILED: The bucket you tried to delete is not empty
```

**Fix:** Empty the S3 bucket first, then retry:

```bash
aws s3 rm s3://<bucket-name> --recursive
aws cloudformation delete-stack --stack-name openai-sagemaker-stack --region eu-north-1
```

---

### API Gateway / Lambda Errors

#### 504 Gateway Timeout

**Cause:** API Gateway has a 29-second hard timeout for HTTP APIs. If SageMaker takes longer to respond (common on cold start), the request times out.

**Fix:** Retry the request. The first request after deployment is the slowest because the model needs a "warm-up" inference. Subsequent requests should complete in 5-15 seconds.

#### Internal Server Error (500)

**Cause:** Lambda function threw an unhandled exception.

**Fix:** Check Lambda logs:

```bash
aws logs tail /aws/lambda/openai-sagemaker-stack-openai-proxy --since 5m
```

Or in the [Lambda Console](https://eu-north-1.console.aws.amazon.com/lambda/home?region=eu-north-1#/functions) > click your function > **Monitor** tab > **View CloudWatch logs**.

---

### EC2 / OpenWebUI Errors

#### OpenWebUI Page Does Not Load

**Causes (in order of likelihood):**

1. **EC2 is still setting up** -- wait 3-5 minutes after stack completion
2. **Docker failed to start** -- connect via Session Manager and check `sudo docker ps`
3. **Security group blocks port 80** -- verify the security group allows inbound HTTP

**Check setup progress:**

```bash
# Via Session Manager (no SSH key needed)
# EC2 Console > select instance > Connect > Session Manager tab > Connect

sudo cat /var/log/cloud-init-output.log | tail -20
sudo docker ps
sudo docker-compose -f /opt/openwebui/docker-compose.yml logs
```

#### OpenWebUI Loads but No Models Available

**Cause:** OpenWebUI cannot reach the API Gateway endpoint.

**Fix:** Verify the `OPENAI_API_BASE_URL` environment variable inside the Docker container:

```bash
sudo docker exec openwebui env | grep OPENAI
```

The value should match your API Gateway URL with `/v1` appended: `https://abc123.execute-api.eu-north-1.amazonaws.com/v1`.

---

### Useful AWS Console Links (eu-north-1)

| Service | Direct Link |
|---------|------------|
| CloudFormation Stacks | [eu-north-1.console.aws.amazon.com/cloudformation/home?region=eu-north-1#/stacks](https://eu-north-1.console.aws.amazon.com/cloudformation/home?region=eu-north-1#/stacks) |
| SageMaker Endpoints | [eu-north-1.console.aws.amazon.com/sagemaker/home?region=eu-north-1#/endpoints](https://eu-north-1.console.aws.amazon.com/sagemaker/home?region=eu-north-1#/endpoints) |
| Lambda Functions | [eu-north-1.console.aws.amazon.com/lambda/home?region=eu-north-1#/functions](https://eu-north-1.console.aws.amazon.com/lambda/home?region=eu-north-1#/functions) |
| API Gateway APIs | [eu-north-1.console.aws.amazon.com/apigateway/home?region=eu-north-1#/apis](https://eu-north-1.console.aws.amazon.com/apigateway/home?region=eu-north-1#/apis) |
| EC2 Instances | [eu-north-1.console.aws.amazon.com/ec2/home?region=eu-north-1#Instances](https://eu-north-1.console.aws.amazon.com/ec2/home?region=eu-north-1#Instances) |
| Service Quotas (SageMaker) | [eu-north-1.console.aws.amazon.com/servicequotas/home/services/sagemaker/quotas](https://eu-north-1.console.aws.amazon.com/servicequotas/home/services/sagemaker/quotas) |
| CloudWatch Logs | [eu-north-1.console.aws.amazon.com/cloudwatch/home?region=eu-north-1#logsV2:log-groups](https://eu-north-1.console.aws.amazon.com/cloudwatch/home?region=eu-north-1#logsV2:log-groups) |
| VPC Subnets | [eu-north-1.console.aws.amazon.com/vpc/home?region=eu-north-1#subnets:](https://eu-north-1.console.aws.amazon.com/vpc/home?region=eu-north-1#subnets:) |
