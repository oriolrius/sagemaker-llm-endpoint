# SageMaker vLLM + OpenAI API + OpenWebUI

Deploy a HuggingFace model on AWS SageMaker with vLLM, exposed via an OpenAI-compatible API, with OpenWebUI for a chat interface.

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌────────────┐     ┌─────────────────┐
│  OpenWebUI  │────▶│ API Gateway  │────▶│   Lambda   │────▶│ SageMaker vLLM  │
│  (EC2)      │     │ (HTTP API)   │     │  (proxy)   │     │   Endpoint      │
└─────────────┘     └──────────────┘     └────────────┘     └─────────────────┘
     ▲                    ▲
     │                    │
     └── Browser ─────────┴── API Clients (curl, Python, etc.)
```

### Components

| Component | Description |
|-----------|-------------|
| **SageMaker Endpoint** | Runs vLLM with your HuggingFace model on GPU |
| **Lambda** | Proxies OpenAI-format requests to SageMaker (handles SigV4 signing) |
| **API Gateway** | Public HTTP API (OpenAI-compatible) |
| **EC2 + OpenWebUI** | Web-based chat interface |

## Quick Start

### Prerequisites

- AWS CLI configured with credentials
- VPC with a public subnet
- GPU quota for ml.g4dn.xlarge (check Service Quotas)
- [uv](https://github.com/astral-sh/uv) (Python package manager) for Lambda packaging

### Deploy

```bash
cd infra/

# Find your VPC and subnet
aws ec2 describe-vpcs --region eu-west-1 \
  --query 'Vpcs[*].[VpcId,Tags[?Key==`Name`].Value|[0]]' --output table

aws ec2 describe-subnets --region eu-west-1 \
  --filters Name=vpc-id,Values=<vpc-id> \
  --query 'Subnets[?MapPublicIpOnLaunch==`true`].[SubnetId,AvailabilityZone]' --output table

# Deploy full stack
./deploy-full-stack.sh \
  --vpc-id vpc-0123456789abcdef0 \
  --subnet-id subnet-0123456789abcdef0
```

Deployment takes ~15-20 minutes (mostly SageMaker endpoint startup).

### Access

After deployment:
- **OpenWebUI**: `http://<ec2-elastic-ip>` (shown in output)
- **API**: `https://<api-id>.execute-api.eu-west-1.amazonaws.com`

### Test API

```bash
# List models
curl https://<api-endpoint>/v1/models

# Chat completion
curl -X POST https://<api-endpoint>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "The future of AI is"}], "max_tokens": 50}'
```

### Cleanup

```bash
cd infra/
./delete-full-stack.sh
```

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--model-id` | distilgpt2 | HuggingFace model ID |
| `--sagemaker-instance` | ml.g4dn.xlarge | GPU instance type |
| `--ec2-instance` | t3a.small | EC2 instance for OpenWebUI |
| `--key-pair` | - | EC2 key pair for SSH access |
| `--stack-name` | openai-sagemaker-stack | CloudFormation stack name |

### Example: Deploy a different model

```bash
./deploy-full-stack.sh \
  --vpc-id vpc-xxx \
  --subnet-id subnet-xxx \
  --model-id gpt2-medium \
  --sagemaker-instance ml.g5.xlarge
```

## Cost

| Resource | Type | Cost |
|----------|------|------|
| SageMaker | ml.g4dn.xlarge | ~$0.74/hour |
| EC2 | t3a.small | ~$0.02/hour |
| API Gateway | HTTP API | ~$1/million requests |

**Total**: ~$0.76/hour (~$550/month if 24/7)

⚠️ **Remember to delete resources when not in use!**

## Notes

### Base Models vs Instruction-Tuned

The default model (distilgpt2) is a **base model**:

| Prompt Type | Works? |
|-------------|--------|
| Text completion: `"The capital of France is"` | ✅ Yes |
| Q&A format: `"Q: What is AI?\nA:"` | ⚠️ Partial |
| Direct questions: `"What is AI?"` | ❌ No |

For chat behavior, use an instruction-tuned model like `meta-llama/Llama-2-7b-chat-hf`.

### Security

⚠️ This setup is for **development/testing**:
- No API authentication
- OpenWebUI auth disabled

For production, add API Gateway authentication and enable OpenWebUI auth.

## Project Structure

```
.
├── scripts/                     # SageMaker deployment & testing tools
│   ├── pyproject.toml           # Python project config (uv)
│   ├── src/sagemaker_tools/
│   │   ├── deploy_vllm.py       # Deploy SageMaker endpoint
│   │   ├── test_openai_endpoint.py  # Test endpoint directly
│   │   ├── test_api_gateway.py  # Test API Gateway
│   │   └── cleanup.py           # Delete SageMaker resources
│   └── README.md
├── lambda/
│   └── openai-proxy/            # Lambda function source
│       ├── pyproject.toml       # Python project config (uv)
│       ├── src/
│       │   ├── index.py         # Lambda entry point
│       │   └── openai_proxy/
│       │       └── handler.py   # Request handlers
│       └── tests/
│           └── test_handler.py  # Unit tests
├── openwebui/                   # OpenWebUI configuration
│   ├── docker-compose.yml       # Docker Compose config
│   ├── setup.sh                 # Setup script
│   └── README.md
├── infra/
│   ├── full-stack.yaml          # CloudFormation template
│   ├── deploy-full-stack.sh     # Deployment script
│   ├── delete-full-stack.sh     # Cleanup script
│   └── README.md
└── README.md                    # This file
```

## Development

### Scripts (SageMaker Tools)

Standalone Python tools for deploying and testing SageMaker endpoints:

```bash
cd scripts/
uv sync

# Deploy standalone SageMaker endpoint (for development/testing)
uv run deploy-vllm

# Test endpoint directly
uv run test-endpoint <endpoint-name>

# Test API Gateway
uv run python -m sagemaker_tools.test_api_gateway https://abc123.execute-api.eu-west-1.amazonaws.com

# Cleanup
uv run cleanup <endpoint-name>
```

### Lambda Function

```bash
cd lambda/openai-proxy
uv sync --dev

# Run tests
uv run pytest

# Lint
uv run ruff check src/ tests/
```

### OpenWebUI (Local)

Run OpenWebUI locally (without CloudFormation):

```bash
cd openwebui/
cp .env.example .env
# Edit .env and set OPENAI_API_BASE_URL

./setup.sh
# Or: docker-compose up -d
```

## License

MIT
