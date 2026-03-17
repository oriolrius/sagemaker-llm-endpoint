# Verification Guide

How to verify that each layer of the deployed stack is working correctly. Work through the checks top-to-bottom -- each layer depends on the ones above it.

**Prerequisites:** AWS CLI configured, `uv` installed, stack deployed via `deploy-full-stack.sh`.

---

## Layer 1: Unit Tests (No AWS Required)

Verify the Lambda proxy code logic before touching anything in the cloud.

```bash
cd lambda/openai-proxy
uv sync --dev
uv run pytest -v
```

**Expected:** All tests pass (14 tests covering routing, CORS, JSON parsing, response formatting, error handling). SageMaker calls are mocked -- this only validates code logic.

```
tests/test_handler.py::TestCreateResponse::test_basic_response PASSED
tests/test_handler.py::TestCreateResponse::test_custom_headers PASSED
...
============================== 14 passed ==============================
```

You can also check code quality:

```bash
uv run ruff check src/ tests/
uv run ruff format --check src/ tests/
```

---

## Layer 2: AWS Credentials

Every subsequent check requires valid AWS credentials.

```bash
aws sts get-caller-identity
```

**Expected:** Returns your account ID, ARN, and user ID.

```json
{
    "UserId": "AROA...:your-session-name",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/..."
}
```

**If this fails:** Your credentials are missing or expired. See [DEBUGGING.md - Credential Issues](DEBUGGING.md#credential-issues).

---

## Layer 3: CloudFormation Stack Status

Confirm the stack deployed successfully and all resources were created.

```bash
aws cloudformation describe-stacks \
  --stack-name openai-sagemaker-stack \
  --region eu-west-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

**Expected:** `CREATE_COMPLETE`

To see all stack outputs (API URL, EC2 IP, endpoint name):

```bash
aws cloudformation describe-stacks \
  --stack-name openai-sagemaker-stack \
  --region eu-west-1 \
  --query 'Stacks[0].Outputs[*].[OutputKey,OutputValue]' \
  --output table
```

Save these values -- you will need them for the checks below:

| Output | Variable | Example |
|--------|----------|---------|
| `ApiGatewayEndpoint` | `API_URL` | `https://abc123.execute-api.eu-west-1.amazonaws.com` |
| `SageMakerEndpointName` | `ENDPOINT_NAME` | `openai-sagemaker-stack-vllm-endpoint` |
| `EC2PublicIP` | `EC2_IP` | `54.123.45.67` |
| `OpenWebUIUrl` | `WEBUI_URL` | `http://54.123.45.67` |

For convenience, export them:

```bash
STACK_NAME="openai-sagemaker-stack"
REGION="eu-west-1"

API_URL=$(aws cloudformation describe-stacks --stack-name $STACK_NAME --region $REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiGatewayEndpoint`].OutputValue' --output text)

ENDPOINT_NAME=$(aws cloudformation describe-stacks --stack-name $STACK_NAME --region $REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`SageMakerEndpointName`].OutputValue' --output text)

EC2_IP=$(aws cloudformation describe-stacks --stack-name $STACK_NAME --region $REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`EC2PublicIP`].OutputValue' --output text)

echo "API_URL=$API_URL"
echo "ENDPOINT_NAME=$ENDPOINT_NAME"
echo "EC2_IP=$EC2_IP"
```

---

## Layer 4: SageMaker Endpoint

The SageMaker endpoint is the slowest resource to provision (15-20 minutes). It must be `InService` before any inference works.

### Check Status

```bash
aws sagemaker describe-endpoint \
  --endpoint-name $ENDPOINT_NAME \
  --region $REGION \
  --query '{Status: EndpointStatus, Name: EndpointName}'
```

**Expected:**

```json
{
    "Status": "InService",
    "Name": "openai-sagemaker-stack-vllm-endpoint"
}
```

| Status | Meaning | Action |
|--------|---------|--------|
| `InService` | Ready | Continue to the next check |
| `Creating` | Still provisioning | Wait (up to 20 minutes) |
| `Failed` | Something broke | See [DEBUGGING.md - SageMaker Endpoint Failed](DEBUGGING.md#sagemaker-endpoint-failed) |
| `RollingBack` | Update failed | Check CloudWatch logs |

### Invoke Directly (Bypass Lambda and API Gateway)

This tests the model container in isolation using the legacy `inputs` + `parameters` format:

```bash
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name $ENDPOINT_NAME \
  --content-type application/json \
  --body '{"inputs": "The capital of France is", "parameters": {"max_new_tokens": 30}}' \
  --region $REGION \
  /dev/stdout
```

**Expected:** A JSON response containing generated text. The model continues the prompt.

```json
[{"generated_text": "The capital of France is Paris. The city of..."}]
```

Or use the test script for a more thorough check (tests OpenAI chat format, streaming, and legacy format):

```bash
cd scripts
uv sync
uv run test-endpoint $ENDPOINT_NAME
```

**Expected:** All three tests print `[PASS]`.

---

## Layer 5: Lambda Function

The Lambda translates OpenAI-format requests into SageMaker invocations. Verify it was deployed correctly.

### Check Lambda Exists and Has Correct Configuration

```bash
aws lambda get-function-configuration \
  --function-name ${STACK_NAME}-openai-proxy \
  --region $REGION \
  --query '{Runtime: Runtime, Handler: Handler, Timeout: Timeout, MemorySize: MemorySize, Env: Environment.Variables}'
```

**Expected:**

```json
{
    "Runtime": "python3.11",
    "Handler": "index.lambda_handler",
    "Timeout": 60,
    "MemorySize": 256,
    "Env": {
        "SAGEMAKER_ENDPOINT_NAME": "openai-sagemaker-stack-vllm-endpoint",
        "AWS_REGION_NAME": "eu-west-1"
    }
}
```

Key things to verify:
- `SAGEMAKER_ENDPOINT_NAME` matches the actual endpoint name from Layer 4
- `Timeout` is 60 seconds (enough for SageMaker cold starts)

### Invoke Lambda Directly (Bypass API Gateway)

```bash
aws lambda invoke \
  --function-name ${STACK_NAME}-openai-proxy \
  --region $REGION \
  --payload '{"requestContext":{"http":{"method":"GET"}},"rawPath":"/v1/models"}' \
  /dev/stdout
```

**Expected:** The Lambda returns the models list:

```json
{"statusCode": 200, "headers": {...}, "body": "{\"object\": \"list\", \"data\": [...]}"}
```

---

## Layer 6: API Gateway (Full Chain)

This tests the entire path: API Gateway -> Lambda -> SageMaker.

### Test 1: List Models (GET)

```bash
curl -s "$API_URL/v1/models" | python3 -m json.tool
```

**Expected:**

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

### Test 2: Chat Completion (POST)

```bash
curl -s -X POST "$API_URL/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "The future of artificial intelligence is"}], "max_tokens": 50}' \
  | python3 -m json.tool
```

**Expected:** A response with generated text in `choices[0].message.content`:

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
                "content": "...generated text..."
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

### Test 3: CORS Preflight (OPTIONS)

```bash
curl -s -X OPTIONS "$API_URL/v1/chat/completions" \
  -H "Origin: http://localhost" \
  -H "Access-Control-Request-Method: POST" \
  -D - -o /dev/null 2>&1 | head -10
```

**Expected:** `200` status with CORS headers (`Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`).

### Automated Test Script

For a comprehensive test of all three API endpoints:

```bash
cd scripts
uv sync
uv run python -m sagemaker_tools.test_api_gateway "$API_URL"
```

**Expected:** All tests print `[PASS]` and the script finishes with `ALL TESTS PASSED!`.

---

## Layer 7: OpenWebUI

The web chat interface running on EC2, connecting to the API Gateway.

### Check EC2 Instance Is Running

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=${STACK_NAME}-openwebui" \
  --region $REGION \
  --query 'Reservations[0].Instances[0].{State: State.Name, IP: PublicIpAddress, ID: InstanceId}' \
  --output table
```

**Expected:** State is `running` with a public IP.

### Check OpenWebUI Responds

```bash
curl -s -o /dev/null -w "%{http_code}" "http://$EC2_IP"
```

**Expected:** `200` (the page loads). If you get `000` (connection refused), wait 3-5 minutes after stack completion -- Docker is still starting.

### Open in Browser

Navigate to `http://<EC2_IP>` in your browser. You should see the OpenWebUI chat interface.

1. The model `openai-sagemaker-stack-vllm-endpoint` appears in the model selector
2. Type a prompt like `The most important invention is` and press Enter
3. You get a generated response within a few seconds

### Check Docker On the EC2 Instance

If OpenWebUI does not load, SSH into the instance (requires a key pair) or use Session Manager:

```bash
# Via SSH
ssh -i <your-key.pem> ec2-user@$EC2_IP

# Then check
sudo docker ps                    # Container should be running
sudo docker logs openwebui        # Check for errors
cat /var/log/cloud-init-output.log | tail -30  # Check UserData setup
```

Via Session Manager (no SSH key needed):
1. Open EC2 Console > select the instance > Connect > Session Manager > Connect
2. Run the commands above

---

## Quick Summary Checklist

Run through this table to confirm everything works:

| # | Check | Command | Expected |
|---|-------|---------|----------|
| 1 | Unit tests | `cd lambda/openai-proxy && uv run pytest -v` | All pass |
| 2 | AWS credentials | `aws sts get-caller-identity` | Returns account info |
| 3 | Stack status | `aws cloudformation describe-stacks --stack-name $STACK_NAME --query 'Stacks[0].StackStatus'` | `CREATE_COMPLETE` |
| 4 | Endpoint status | `aws sagemaker describe-endpoint --endpoint-name $ENDPOINT_NAME --query EndpointStatus` | `InService` |
| 5 | SageMaker direct | `aws sagemaker-runtime invoke-endpoint --endpoint-name $ENDPOINT_NAME --body '{"inputs":"Hello"}' /dev/stdout` | Returns JSON |
| 6 | Lambda direct | `aws lambda invoke --function-name ${STACK_NAME}-openai-proxy --payload '...' /dev/stdout` | Returns 200 |
| 7 | API GET /v1/models | `curl $API_URL/v1/models` | Model list JSON |
| 8 | API POST /v1/chat/completions | `curl -X POST $API_URL/v1/chat/completions -d '...'` | Generated text |
| 9 | OpenWebUI HTTP | `curl -o /dev/null -w "%{http_code}" http://$EC2_IP` | `200` |
| 10 | OpenWebUI browser | Open `http://$EC2_IP` | Chat interface loads |

If any check fails, see [DEBUGGING.md](DEBUGGING.md) for diagnosis and resolution.
