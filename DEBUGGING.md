# Debugging Guide

How to diagnose and resolve issues with the deployed stack. Organized by component -- jump to the section matching your problem.

---

## Table of Contents

- [Quick Diagnostic Table](#quick-diagnostic-table)
- [Credential Issues](#credential-issues)
- [CloudFormation Failures](#cloudformation-failures)
- [SageMaker Endpoint](#sagemaker-endpoint)
- [Lambda Function](#lambda-function)
- [API Gateway](#api-gateway)
- [OpenWebUI (EC2)](#openwebui-ec2)
- [CloudWatch Logs Reference](#cloudwatch-logs-reference)
- [Useful AWS Console Links](#useful-aws-console-links)

---

## Quick Diagnostic Table

Start here. Find your symptom and follow the pointer.

| Symptom | Likely Component | Jump To |
|---------|-----------------|---------|
| `ExpiredTokenException` or `Unable to locate credentials` | AWS credentials | [Credential Issues](#credential-issues) |
| Stack shows `ROLLBACK_IN_PROGRESS` or `ROLLBACK_COMPLETE` | CloudFormation | [CloudFormation Failures](#cloudformation-failures) |
| `ResourceLimitExceeded` during deployment | GPU quota | [Quota Exceeded](#quota-exceeded) |
| Endpoint stuck in `Creating` for >25 minutes | SageMaker | [Endpoint Stuck Creating](#endpoint-stuck-in-creating) |
| Endpoint status is `Failed` | SageMaker | [SageMaker Endpoint Failed](#sagemaker-endpoint-failed) |
| `curl /v1/models` returns empty or connection refused | API Gateway / Lambda | [API Gateway](#api-gateway) |
| `curl /v1/chat/completions` returns 500 | Lambda -> SageMaker | [Lambda Returns 500](#lambda-returns-500) |
| `curl /v1/chat/completions` returns 504 timeout | API Gateway timeout | [API Gateway Timeout](#api-gateway-timeout) |
| OpenWebUI page does not load | EC2 / Docker | [OpenWebUI Not Loading](#openwebui-not-loading) |
| OpenWebUI loads but shows no models | OpenWebUI config | [OpenWebUI No Models](#openwebui-shows-no-models) |
| OpenWebUI loads but responses fail | Full chain | [OpenWebUI Responses Fail](#openwebui-responses-fail) |

---

## Credential Issues

### ExpiredTokenException

```
An error occurred (ExpiredTokenException) when calling the ... operation:
The security token included in the request is expired
```

**Cause:** AWS session tokens from Innovation Sandbox or SSO expire after 1-12 hours.

**Fix:** Get fresh credentials and reconfigure:

```bash
# Option A: Environment variables
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

# Option B: AWS CLI config
aws configure set aws_access_key_id <KEY>
aws configure set aws_secret_access_key <SECRET>
aws configure set aws_session_token <TOKEN>

# Verify
aws sts get-caller-identity
```

### Unable to Locate Credentials

```
Unable to locate credentials. You can configure credentials by running "aws configure".
```

**Cause:** No AWS credentials configured at all.

**Fix:** Configure credentials using one of the methods above, or via SSO:

```bash
aws configure sso
```

### AccessDeniedException

```
User: arn:aws:sts::123456789012:assumed-role/... is not authorized to perform: sagemaker:InvokeEndpoint
```

**Cause:** Your IAM role lacks the required permission. The error message tells you the exact missing action.

**Fix:** Use a role with broader permissions (e.g., `AdministratorAccess` for sandbox environments) or ask your administrator to add the specific permission listed in the error.

### Region Mismatch

```
Could not connect to the endpoint URL: "https://sagemaker.us-east-1.amazonaws.com/"
```

**Cause:** AWS CLI is configured for a different region than where your resources are deployed.

**Fix:** Always pass `--region eu-west-1` or set the default:

```bash
export AWS_DEFAULT_REGION=eu-west-1
```

---

## CloudFormation Failures

### Finding the Root Cause

When a stack fails, CloudFormation rolls back all resources. The key is finding the **first** `CREATE_FAILED` event:

```bash
aws cloudformation describe-stack-events \
  --stack-name openai-sagemaker-stack \
  --region eu-west-1 \
  --query "StackEvents[?ResourceStatus=='CREATE_FAILED'].[LogicalResourceId,ResourceStatusReason]" \
  --output table
```

### Quota Exceeded

```
ResourceLimitExceeded: The account-level service limit 'ml.g4dn.xlarge for endpoint usage' is 0 Instances
```

**Cause:** Your account has zero GPU quota for the requested instance type.

**Fix:** Request a quota increase. You cannot deploy until it is approved.

```bash
# Check current quota
aws service-quotas list-service-quotas \
  --service-code sagemaker --region eu-west-1 \
  --query "Quotas[?contains(QuotaName, 'ml.g4dn.xlarge') && contains(QuotaName, 'endpoint')].[QuotaName,Value]" \
  --output table

# Request increase
QUOTA_CODE=$(aws service-quotas list-service-quotas \
  --service-code sagemaker --region eu-west-1 \
  --query "Quotas[?contains(QuotaName, 'ml.g4dn.xlarge') && contains(QuotaName, 'endpoint')].QuotaCode" \
  --output text)

aws service-quotas request-service-quota-increase \
  --service-code sagemaker --quota-code "$QUOTA_CODE" \
  --desired-value 1 --region eu-west-1

# Monitor status
aws service-quotas list-requested-service-quota-change-history \
  --service-code sagemaker --region eu-west-1 \
  --query "RequestedQuotas[?contains(QuotaName, 'g4dn')].[QuotaName,Status,DesiredValue]" \
  --output table
```

Approval times: sandbox accounts often within minutes; production accounts 1-3 business days.

### Stack in ROLLBACK_COMPLETE

A stack in `ROLLBACK_COMPLETE` state blocks redeployment with the same name. Delete it first:

```bash
aws cloudformation delete-stack \
  --stack-name openai-sagemaker-stack \
  --region eu-west-1

aws cloudformation wait stack-delete-complete \
  --stack-name openai-sagemaker-stack \
  --region eu-west-1
```

Then fix the underlying issue and redeploy.

### Stack Deletion Fails

```
DELETE_FAILED: The bucket you tried to delete is not empty
```

**Fix:** Empty the S3 bucket, then retry:

```bash
aws s3 rm s3://<bucket-name> --recursive --region eu-west-1
aws cloudformation delete-stack --stack-name openai-sagemaker-stack --region eu-west-1
```

### IAM Role Already Exists

```
<role-name> already exists
```

**Cause:** A previous failed deployment left behind IAM roles that CloudFormation could not clean up.

**Fix:** Delete the orphaned role manually:

```bash
# List and detach policies first
aws iam list-attached-role-policies --role-name <role-name>
aws iam detach-role-policy --role-name <role-name> --policy-arn <policy-arn>
aws iam delete-role --role-name <role-name>
```

Then redeploy.

---

## SageMaker Endpoint

### Checking Endpoint Status

```bash
aws sagemaker describe-endpoint \
  --endpoint-name openai-sagemaker-stack-vllm-endpoint \
  --region eu-west-1 \
  --query '{Status: EndpointStatus, FailureReason: FailureReason}'
```

### SageMaker Endpoint Failed

When the endpoint status is `Failed`, get the reason:

```bash
aws sagemaker describe-endpoint \
  --endpoint-name openai-sagemaker-stack-vllm-endpoint \
  --region eu-west-1 \
  --query 'FailureReason' --output text
```

Then check the container logs:

```bash
aws logs tail /aws/sagemaker/Endpoints/openai-sagemaker-stack-vllm-endpoint \
  --region eu-west-1 --since 1h
```

#### Common Failure Reasons

**CUDA Out of Memory:**

```
torch.cuda.OutOfMemoryError: CUDA out of memory
```

The model is too large for the GPU. The T4 on `ml.g4dn.xlarge` has 16 GB. Models larger than ~7B parameters in fp16 will not fit.

**Fix:** Use a smaller model or a larger instance:

```bash
# Redeploy with a smaller model
./deploy-full-stack.sh --vpc-id vpc-xxx --subnet-id subnet-xxx --model-id distilgpt2

# Or request quota for a larger instance and redeploy
./deploy-full-stack.sh --vpc-id vpc-xxx --subnet-id subnet-xxx --sagemaker-instance ml.g5.xlarge
```

**Model Not Found:**

```
OSError: <model-id> is not a valid model identifier listed on 'https://huggingface.co/models'
```

The `HuggingFaceModelId` parameter contains an invalid model ID.

**Fix:** Verify the model exists on HuggingFace Hub, then redeploy with the correct ID.

**ECR Image Pull Failure:**

```
CannotPullContainerError: pull image manifest has been retried
```

The SageMaker execution role cannot pull the DJL-LMI container image from ECR.

**Fix:** Verify the role has ECR permissions. If using `ExternalSageMakerRoleArn`, ensure that role includes:
- `ecr:GetAuthorizationToken`
- `ecr:BatchCheckLayerAvailability`
- `ecr:GetDownloadUrlForLayer`
- `ecr:BatchGetImage`

### Endpoint Stuck in Creating

Normal creation takes 15-20 minutes. If it exceeds 25 minutes:

1. Check if logs are being produced:

```bash
aws logs tail /aws/sagemaker/Endpoints/openai-sagemaker-stack-vllm-endpoint \
  --region eu-west-1 --follow
```

2. If **no logs appear** after 10 minutes, the container failed to start. This usually means an ECR pull failure or IAM misconfiguration.

3. If logs show **model downloading**, it is still working. Large models can take 10+ minutes to download.

4. If logs show errors, see the failure reasons above.

### Testing the Endpoint Directly

Bypass all other components and talk to SageMaker directly:

```bash
# Legacy format (inputs + parameters)
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name openai-sagemaker-stack-vllm-endpoint \
  --content-type application/json \
  --body '{"inputs": "Hello world", "parameters": {"max_new_tokens": 30}}' \
  --region eu-west-1 \
  /dev/stdout

# Full test suite
cd scripts && uv sync
uv run test-endpoint openai-sagemaker-stack-vllm-endpoint
```

If the direct invocation fails with `ModelError`, the model itself is broken. Check the container logs.

If direct invocation works but the API Gateway chain fails, the problem is in Lambda or API Gateway.

---

## Lambda Function

### Lambda Returns 500

A 500 from the API usually means the Lambda threw an exception.

**Step 1:** Check Lambda logs:

```bash
aws logs tail /aws/lambda/openai-sagemaker-stack-openai-proxy \
  --region eu-west-1 --since 5m
```

**Step 2:** Look for the error pattern:

| Log Pattern | Cause | Fix |
|-------------|-------|-----|
| `AccessDeniedException...InvokeEndpoint` | Lambda role cannot invoke SageMaker | Check `SageMakerInvokePolicy` in the CloudFormation template targets the correct endpoint ARN |
| `Could not find endpoint` | Wrong endpoint name in env var | Verify `SAGEMAKER_ENDPOINT_NAME` environment variable matches the actual endpoint |
| `Connection refused` or `timeout` | Endpoint not ready | Wait for endpoint to be `InService` |
| `ImportError` or `ModuleNotFoundError` | Lambda package missing dependencies | Rebuild and redeploy the Lambda package |

**Step 3:** Verify the Lambda configuration:

```bash
aws lambda get-function-configuration \
  --function-name openai-sagemaker-stack-openai-proxy \
  --region eu-west-1 \
  --query '{Timeout: Timeout, Env: Environment.Variables}'
```

Confirm `SAGEMAKER_ENDPOINT_NAME` matches the actual endpoint name.

### Lambda Timeout

The Lambda has a 60-second timeout. If SageMaker takes longer (common on first cold start after deployment):

```bash
# Check if the Lambda timed out
aws logs filter-log-events \
  --log-group-name /aws/lambda/openai-sagemaker-stack-openai-proxy \
  --filter-pattern "Task timed out" \
  --region eu-west-1
```

**Fix:** Retry the request. The first invocation after deployment warms up the model. Subsequent requests should complete in 5-15 seconds.

If timeouts persist, increase the Lambda timeout in `infra/full-stack.yaml` (the `Timeout` property on the `LambdaFunction` resource).

### Invoke Lambda Directly

Test the Lambda in isolation (bypasses API Gateway):

```bash
# Test GET /v1/models
aws lambda invoke \
  --function-name openai-sagemaker-stack-openai-proxy \
  --region eu-west-1 \
  --cli-binary-format raw-in-base64-out \
  --payload '{"requestContext":{"http":{"method":"GET"}},"rawPath":"/v1/models"}' \
  /dev/stdout

# Test POST /v1/chat/completions
aws lambda invoke \
  --function-name openai-sagemaker-stack-openai-proxy \
  --region eu-west-1 \
  --cli-binary-format raw-in-base64-out \
  --payload '{"requestContext":{"http":{"method":"POST"}},"rawPath":"/v1/chat/completions","body":"{\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}],\"max_tokens\":20}"}' \
  /dev/stdout
```

If direct invocation works but API Gateway does not, the problem is in API Gateway routing.

---

## API Gateway

### No Response / Connection Refused

**Check the API exists:**

```bash
aws apigatewayv2 get-apis \
  --region eu-west-1 \
  --query "Items[?contains(Name, 'openai-sagemaker-stack')].[Name,ApiEndpoint]" \
  --output table
```

If it returns nothing, the API Gateway was not created. Check the CloudFormation stack.

### Wrong URL

API Gateway HTTP APIs have this URL format:

```
https://<api-id>.execute-api.<region>.amazonaws.com
```

There is **no stage prefix** in the URL (HTTP APIs with `$default` stage). Do not add `/prod` or `/default`.

- Correct: `https://abc123.execute-api.eu-west-1.amazonaws.com/v1/models`
- Wrong: `https://abc123.execute-api.eu-west-1.amazonaws.com/prod/v1/models`

### 403 Forbidden

**Cause:** The URL is wrong, or the route does not exist.

**Check routes:**

```bash
API_ID=$(aws apigatewayv2 get-apis --region eu-west-1 \
  --query "Items[?contains(Name, 'openai-sagemaker-stack')].ApiId" --output text)

aws apigatewayv2 get-routes \
  --api-id $API_ID \
  --region eu-west-1 \
  --query 'Items[*].RouteKey' --output table
```

**Expected routes:**
- `GET /v1/models`
- `POST /v1/chat/completions`
- `POST /v1/completions`

### API Gateway Timeout

```
HTTP 504 Gateway Timeout
```

**Cause:** API Gateway HTTP APIs have a hard 29-second timeout. If the Lambda + SageMaker combined latency exceeds this, the request times out.

**Common triggers:**
- First request after deployment (SageMaker model warm-up)
- Large `max_tokens` value causing slow generation
- Lambda cold start + SageMaker cold start combined

**Fix:** Retry the request. First invocations are the slowest. If timeouts persist:

1. Check SageMaker endpoint is `InService`
2. Test SageMaker directly (see [Testing the Endpoint Directly](#testing-the-endpoint-directly))
3. Reduce `max_tokens` in your request

### CORS Issues

If a browser-based client gets CORS errors:

```bash
# Test CORS preflight
curl -v -X OPTIONS "$API_URL/v1/chat/completions" \
  -H "Origin: http://localhost" \
  -H "Access-Control-Request-Method: POST" 2>&1 | grep -i "access-control"
```

**Expected headers:**
```
access-control-allow-origin: *
access-control-allow-methods: GET, POST, OPTIONS
access-control-allow-headers: Content-Type, Authorization
```

CORS is configured at two levels:
1. API Gateway `CorsConfiguration` in the CloudFormation template
2. Lambda response headers in `handler.py` (`Access-Control-Allow-Origin: *`)

---

## OpenWebUI (EC2)

### OpenWebUI Not Loading

The EC2 instance takes 3-5 minutes after the stack completes to finish installing Docker and starting the container.

**Step 1:** Confirm the instance is running:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=openai-sagemaker-stack-openwebui" \
  --region eu-west-1 \
  --query 'Reservations[0].Instances[0].{State: State.Name, IP: PublicIpAddress}' \
  --output table
```

**Step 2:** Check if port 80 is open:

```bash
curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 "http://<EC2_IP>"
```

- `200`: OpenWebUI is running
- `000`: Connection refused -- Docker not started yet, or security group blocks port 80
- `No route to host`: EC2 not reachable -- check security group and Elastic IP

**Step 3:** Connect to the instance and inspect:

```bash
# Via SSH (if key pair was provided)
ssh -i <your-key.pem> ec2-user@<EC2_IP>

# Or via Session Manager (no key needed)
# EC2 Console > select instance > Connect > Session Manager > Connect
```

Once connected:

```bash
# Check if Docker is running
sudo systemctl status docker

# Check if the container is running
sudo docker ps

# Check container logs
sudo docker logs openwebui

# Check the full setup log
sudo cat /var/log/cloud-init-output.log
```

**Step 4:** Common issues found in logs:

| Log / Error | Cause | Fix |
|-------------|-------|-----|
| `docker: command not found` | Docker install failed | Run `sudo dnf install -y docker && sudo systemctl start docker` |
| `docker-compose: command not found` | docker-compose not installed | Run the install command from `setup.sh` manually |
| Container exits immediately | Bad environment variable | Check `docker logs openwebui` for specifics |
| No log file at all | UserData did not run | Check `/var/log/cloud-init.log` for errors |

### OpenWebUI Shows No Models

**Cause:** OpenWebUI cannot reach the API Gateway, or the `OPENAI_API_BASE_URL` is wrong.

**Check the configured URL:**

```bash
# On the EC2 instance
sudo docker exec openwebui env | grep OPENAI
```

**Expected:**
```
OPENAI_API_BASE_URL=https://abc123.execute-api.eu-west-1.amazonaws.com/v1
```

Note the `/v1` suffix -- the `docker-compose.yml` appends it automatically.

**Test from the EC2 instance:**

```bash
# On the EC2 instance
curl -s https://abc123.execute-api.eu-west-1.amazonaws.com/v1/models
```

If this fails from EC2 but works from your local machine, the EC2 security group may be blocking outbound HTTPS. Check the egress rules.

### OpenWebUI Responses Fail

OpenWebUI loads and shows the model, but sending a message returns an error.

**Check the full chain from the EC2 instance:**

```bash
# On the EC2 instance
curl -s -X POST https://abc123.execute-api.eu-west-1.amazonaws.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Hello"}],"max_tokens":20}'
```

If this returns an error:
- `500`: Lambda/SageMaker problem -- see [Lambda Returns 500](#lambda-returns-500)
- `504`: Timeout -- see [API Gateway Timeout](#api-gateway-timeout)
- `Connection refused`: Network issue from EC2

**Restart the container** if the config was changed:

```bash
# On the EC2 instance
cd /opt/openwebui
sudo docker-compose down
sudo docker-compose up -d
```

---

## CloudWatch Logs Reference

All components write logs to CloudWatch. Here is where to find them.

### Log Groups

| Component | Log Group | What It Contains |
|-----------|-----------|-----------------|
| SageMaker Endpoint | `/aws/sagemaker/Endpoints/<endpoint-name>` | Container stdout/stderr, model loading, inference errors |
| Lambda Function | `/aws/lambda/<function-name>` | Request/response details, Python exceptions, timeouts |
| EC2 UserData | Accessible via `cloud-init-output.log` on the instance | Docker install, setup script output |

### Tailing Logs

```bash
# SageMaker endpoint logs (model container)
aws logs tail /aws/sagemaker/Endpoints/openai-sagemaker-stack-vllm-endpoint \
  --region eu-west-1 --follow

# Lambda function logs
aws logs tail /aws/lambda/openai-sagemaker-stack-openai-proxy \
  --region eu-west-1 --follow

# Lambda errors only
aws logs filter-log-events \
  --log-group-name /aws/lambda/openai-sagemaker-stack-openai-proxy \
  --filter-pattern "ERROR" \
  --region eu-west-1
```

### Searching Logs

```bash
# Find all Lambda timeouts
aws logs filter-log-events \
  --log-group-name /aws/lambda/openai-sagemaker-stack-openai-proxy \
  --filter-pattern "Task timed out" \
  --region eu-west-1

# Find SageMaker CUDA errors
aws logs filter-log-events \
  --log-group-name /aws/sagemaker/Endpoints/openai-sagemaker-stack-vllm-endpoint \
  --filter-pattern "CUDA" \
  --region eu-west-1

# Find all errors across a log group (last 1 hour)
aws logs filter-log-events \
  --log-group-name /aws/lambda/openai-sagemaker-stack-openai-proxy \
  --filter-pattern "?ERROR ?Exception ?error ?Traceback" \
  --start-time $(date -d '1 hour ago' +%s000) \
  --region eu-west-1
```

---

## Useful AWS Console Links

All links target `eu-west-1` (Ireland). Replace the region if deploying elsewhere.

| Service | Link |
|---------|------|
| CloudFormation Stacks | [eu-west-1.console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks](https://eu-west-1.console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks) |
| SageMaker Endpoints | [eu-west-1.console.aws.amazon.com/sagemaker/home?region=eu-west-1#/endpoints](https://eu-west-1.console.aws.amazon.com/sagemaker/home?region=eu-west-1#/endpoints) |
| Lambda Functions | [eu-west-1.console.aws.amazon.com/lambda/home?region=eu-west-1#/functions](https://eu-west-1.console.aws.amazon.com/lambda/home?region=eu-west-1#/functions) |
| API Gateway APIs | [eu-west-1.console.aws.amazon.com/apigateway/home?region=eu-west-1#/apis](https://eu-west-1.console.aws.amazon.com/apigateway/home?region=eu-west-1#/apis) |
| EC2 Instances | [eu-west-1.console.aws.amazon.com/ec2/home?region=eu-west-1#Instances](https://eu-west-1.console.aws.amazon.com/ec2/home?region=eu-west-1#Instances) |
| CloudWatch Log Groups | [eu-west-1.console.aws.amazon.com/cloudwatch/home?region=eu-west-1#logsV2:log-groups](https://eu-west-1.console.aws.amazon.com/cloudwatch/home?region=eu-west-1#logsV2:log-groups) |
| Service Quotas (SageMaker) | [eu-west-1.console.aws.amazon.com/servicequotas/home/services/sagemaker/quotas](https://eu-west-1.console.aws.amazon.com/servicequotas/home/services/sagemaker/quotas) |
