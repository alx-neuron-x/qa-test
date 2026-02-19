# Running a Validator

This guide covers everything you need to set up and run a QA-Subnet validator.

---

## Hardware Requirements

The validator needs to run Docker containers for mutation testing and maintain a persistent connection to the Bittensor blockchain. The recommended setup is a VPS or dedicated server with at least 4 CPU cores, 8 GB of RAM, 50 GB of SSD storage, and a stable internet connection. The minimum viable configuration is 2 CPU cores and 4 GB of RAM, though evaluation rounds will be slower. Docker must be installed and the user must have permission to run containers. Ubuntu 22.04+ is the recommended operating system, though any Linux distribution with Docker support will work.

---

## Installation

Start by cloning the repository and installing the dependencies. Python 3.11 or higher is required.

```bash
git clone https://github.com/your-org/qa-subnet.git
cd qa-subnet

python3.11 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install uv

uv pip install -e .
```

Next, create a Bittensor wallet if you do not already have one. You will need a coldkey (which holds your TAO) and a hotkey (which identifies your miner or validator on-chain).

```bash
btcli wallet new_coldkey --wallet.name my_wallet
btcli wallet new_hotkey --wallet.name my_wallet --wallet.hotkey my_hotkey
```

Register your hotkey on the subnet. This requires a small amount of TAO for the registration fee.

```bash
btcli subnet register --wallet.name my_wallet --wallet.hotkey my_hotkey --netuid 389 --subtensor.network test
```

Copy the environment configuration file and edit it with your settings.

```bash
cp .env .env.backup
```

---

## Prerequisites

The validator requires Docker to run mutation testing sandboxes. Install Docker following the official instructions for your platform, then build the sandbox image.

```bash
cd docker
docker build -t qa-runner -f Dockerfile.runner .
cd ..
```

Verify Docker is working.

```bash
docker run --rm qa-runner echo "Docker is ready"
```

The validator also needs a Chutes.ai API key because it calls each miner's model directly through the Chutes API.

If you don't have a chutes.ai account, you can create one with command:

```bash
chutes register
```
and follow the instructions on the terminal. At the end of the registration process don't forget to back up your fingerprint and credentials.

---

## Configuration

Open the `.env` file and set at minimum these variables:

```env
# Network (required)
NETWORK=test
NETUID=389
SUBTENSOR_NETWORK=test
SUBTENSOR_CHAIN_ENDPOINT=wss://test.chain.opentensor.ai:443

# Chutes.ai (required — the validator calls each miner's model via Chutes)
CHUTES_API_KEY=cpk_your_api_key_here
CHUTES_BASE_URL=https://llm.chutes.ai/v1
```

UID 0 acts as the baseline reference. It must be running as a real miner with a model deployed on Chutes.ai. The validator reads UID 0's model from on-chain data — there is no hardcoded baseline model. Whatever model UID 0 runs becomes the baseline automatically.

The remaining validator settings (`DOCKER_TIMEOUT`, `MAX_DOCKER_CONCURRENT`) have sensible defaults and typically do not need changes. EMA smoothing (warmup alpha = 0.05 for first 20 rounds, then stable alpha = 0.02) and batch size (up to 10 miners per round) are hardcoded in the validator.

### Validator-specific environment variables

```env
DOCKER_TIMEOUT=1200                                       # Max seconds per sandbox run (default: 1200)
# MAX_DOCKER_CONCURRENT=10                                # Max parallel Docker containers (default: 10)
```

### Optional

```env
ENABLE_WANDB=false                                        # Weights & Biases dashboard
WANDB_API_KEY=                                            # W&B key (wandb.ai/authorize)
HF_TOKEN=hf_your_token_here                               # HuggingFace token (avoids rate limits)
```

---

## Starting the Validator

```bash
python neurons/validator.py \
  --netuid 389 \
  --subtensor.network test \
  --wallet.name my_wallet \
  --wallet.hotkey default \
  --logging.debug
```

The validator will synchronize the metagraph, begin generating challenges, select miners, verify their models, call their Chutes endpoints, run mutation testing, and update weights. Each round produces detailed logs showing per-miner scores, EMA updates, and weight distributions. For production deployment, use a process manager like `pm2`, `screen` or `systemd` to keep the validator running.

```bash
pm2 start neurons/validator.py --name qa-validator --interpreter python3 -- \
  --netuid 389 --subtensor.network test \
  --wallet.name my_wallet --wallet.hotkey default
```
