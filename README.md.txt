
# Alchemyst AI - DevOps Internship Assignment
**Candidate:** Ayushi Gautam | ayushigautam1919@gmail.com | github.com/ayusheez

---

## What I Built

This submission deploys the Alchemyst quickstart project - a distributed inference system with a Python worker running gemma-3-270m and a TypeScript worker exposing it via HTTP - across isolated VMs in a private AWS subnet.

---

## Architecture

![Architecture](./architecture.png)

## RPC Call Chain

```
POST /v1/chat/completions
  → http::run_inference_over_http   (caller-worker, TypeScript)
  → inference::get_response         (caller-worker, TypeScript)
  → inference::run_inference        (inference-worker, Python)
  → gemma-3-270m model runs
  → result bubbles back up as JSON
```

---

## Sample curl Command

```bash
curl -X POST http://<VM1_PUBLIC_IP>:3111/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "Say hello in one sentence"}]}'
```

---

## How I Thought Through This

Before touching any cloud console, I read through the entire codebase first. The quickstart has two workers — a Python one that loads the model and exposes inference::run_inference, and a TypeScript one that receives HTTP requests and calls that function via RPC. iii acts as the middleware that lets them talk to each other regardless of language or machine.

The architecture decision was straightforward once I understood that: the TypeScript worker needs a public IP because it's the HTTP entry point, the Python worker doesn't because nothing outside the VPC should talk to it directly. RPC stays internal.

For the VPC design I went with a simple public/private subnet split — public subnet for caller-worker, private subnet for inference-worker, internet gateway only on the public side. Security groups allow port 3111 from anywhere on VM-1, and only allow port 49134 from VM-1's security group on VM-2. Nothing else.

For IaC I chose Terraform because it's the most standard choice for AWS and the most readable for someone reviewing the submission.

The thing I'd do differently with more time: use a NAT Gateway so the private subnet VM can pull dependencies without a public IP, and use systemd units instead of running iii manually so it restarts automatically on crash.

---

## What I Got Working Locally

Okay so I'm on Windows which immediately made things fun. iii CLI doesn't work on Windows so first thing was setting up WSL2. Got Ubuntu running, installed iii, then realized the config.yaml had hardcoded paths to some guy named Anuran's Mac. Fixed that with sed.

Engine started fine — API listening on port 3111, both workers spinning up inside iii's sandbox. You could see them pulling Docker images for Python and Node runtimes. Then the model download started. torch alone is 532MB, then CUDA libraries on top of that — nearly 1.5GB total. My connection timed out halfway through nvidia_cufft.

So the engine is live, workers are running, HTTP endpoint is up. The model just didn't finish loading before the deadline.

---

## How to Redeploy from Scratch

### Prerequisites
- AWS account + CLI configured (aws configure)
- Terraform installed
- SSH key pair created in AWS

### Step 1 — Provision Infrastructure
```bash
cd infrastructure
terraform init
terraform apply -var="key_name=your-key-name"
```

This creates:
- VPC with public + private subnets
- Internet Gateway for public subnet
- Security groups (port 3111 public on VM-1, port 49134 internal only on VM-2)
- 2 EC2 t2.micro instances (Ubuntu 22.04)

### Step 2 — Deploy inference-worker (VM-2, private)
```bash
ssh -i your-key.pem ubuntu@<VM2_PRIVATE_IP>
curl -fsSL https://install.iii.dev/iii/main/install.sh | sh
git clone https://github.com/ayusheez/alchemyst-devops-submission.git
cd alchemyst-devops-submission/workers/inference-worker
pip install -r requirements.txt
cd ../..
iii
```

### Step 3 — Deploy caller-worker (VM-1, public)
```bash
ssh -i your-key.pem ubuntu@<VM1_PUBLIC_IP>
curl -fsSL https://install.iii.dev/iii/main/install.sh | sh
git clone https://github.com/ayusheez/alchemyst-devops-submission.git
cd alchemyst-devops-submission
iii
```

### Step 4 — Test
```bash
curl -X POST http://<VM1_PUBLIC_IP>:3111/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "hello"}]}'
```

---

## Production Hardening

**Security**
- TLS on the public endpoint (Let's Encrypt via Certbot)
- SSH only through a bastion host, never direct from internet
- Secrets in AWS Secrets Manager, not environment variables
- VPC Flow Logs enabled for traffic auditing

**Reliability**
- Application Load Balancer in front of caller-worker
- Auto Scaling Group for the API tier
- Health checks + automatic restarts via systemd
- Model weights on EFS so multiple inference VMs share one copy

**Observability**
- Centralized logging to CloudWatch
- Prometheus + Grafana for latency and throughput metrics
- Alerts when inference latency spikes

---

## If the Model Were 100x Larger

A 270M model runs fine on CPU. At ~27B parameters:

- GPU instances required (AWS g5 or p4d) - cost jumps from ~$0.01/hr to $3-16/hr
- Model weights are 50-100GB+ - store on S3, load at startup, not bundled with code
- Single GPU likely can't hold the full model - need tensor parallelism across GPUs
- Use vLLM or TensorRT-LLM instead of raw transformers for throughput
- Cold start becomes a real problem - keep instances warm or use SageMaker endpoints
- Spot instances with on-demand fallback to manage cost

---

## Challenges Faced

- iii is Linux-only — had to set up WSL2 just to run the CLI on Windows
- config.yaml had the original author's absolute Mac paths hardcoded - broke immediately on any other machine
- torch pulls CUDA libraries even on CPU-only machines - 1.5GB of GPU deps for a 270M model that doesn't need a GPU
- iii sandboxes workers in microVMs and pulls Docker images at startup - adds significant cold start time on first run
- Network timeout killed the pip install at 1.2GB downloaded

The last one is annoying but also kind of the point - these are exactly the problems you hit in real deployments. Dependency size, environment parity, cold starts. Documented them here instead of pretending they didn't happen.
