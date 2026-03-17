# SageMaker Tools

Python tools for deploying and testing SageMaker vLLM endpoints.

## Installation

```bash
cd scripts/

# Install with uv
uv sync

# Or install with pip
pip install -e .
```

## Usage

### Deploy SageMaker Endpoint

Deploy a standalone vLLM endpoint (useful for testing/development outside CloudFormation):

```bash
# Default: Qwen/Qwen2.5-1.5B-Instruct on ml.g4dn.xlarge
uv run deploy-vllm

# Custom model
HF_MODEL_ID=gpt2-medium uv run deploy-vllm

# Different instance type
INSTANCE_TYPE=ml.g5.xlarge uv run deploy-vllm
```

### Test SageMaker Endpoint

Test the endpoint directly with OpenAI-compatible API:

```bash
# Test specific endpoint
uv run test-endpoint vllm-endpoint-20260122-123456

# Auto-detect latest vLLM endpoint
uv run test-endpoint
```

### Test API Gateway

Test the API Gateway + Lambda proxy:

```bash
uv run python -m sagemaker_tools.test_api_gateway https://abc123.execute-api.eu-west-1.amazonaws.com
```

### Cleanup Resources

```bash
# Delete specific endpoint
uv run cleanup vllm-endpoint-20260122-123456

# List all vLLM endpoints
uv run cleanup --list

# Delete all vLLM endpoints
uv run cleanup --all
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `AWS_REGION` | AWS region | `eu-west-1` |
| `HF_MODEL_ID` | HuggingFace model ID | `Qwen/Qwen2.5-1.5B-Instruct` |
| `INSTANCE_TYPE` | SageMaker instance type | `ml.g4dn.xlarge` |
| `SAGEMAKER_ROLE_ARN` | IAM role ARN | Auto-detected |
| `SAGEMAKER_ENDPOINT_NAME` | Endpoint for testing | Auto-detected |

## Development

```bash
# Install dev dependencies
uv sync --dev

# Run tests
uv run pytest

# Lint
uv run ruff check src/
```

## Scripts

| Script | Description |
|--------|-------------|
| `deploy-vllm` | Deploy SageMaker vLLM endpoint |
| `test-endpoint` | Test endpoint with OpenAI API format |
| `cleanup` | Delete SageMaker resources |
