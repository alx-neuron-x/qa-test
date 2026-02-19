# Running a Miner

Mining on QA-Subnet is a three-phase process: fine-tune a model, deploy it to Chutes, and run the miner process. This guide walks through a complete example using a hypothetical miner called `alice` with hotkey `5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY`.

---

## Hardware Requirements

The miner process itself is lightweight since it only maintains a blockchain connection and does not perform inference. A machine with 1 CPU core, 1 GB of RAM, and a stable internet connection is sufficient. The computational cost of the miner is in the fine-tuning phase (done once, before registration) and the Chutes.ai hosting fees for serving the model. Fine-tuning a 7B-parameter model typically requires a GPU with at least 24 GB of VRAM (such as an RTX 3090 or A100) and several hours of training time, depending on the dataset size and training configuration.

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

## Phase 1: Fine-Tune Your Model

The goal is to produce a language model that excels at writing Python unit tests that catch mutations. Start with a strong code-generation base model and fine-tune it on a dataset of (code, high-quality-tests) pairs. Focus your training data on tests that cover edge cases, boundary conditions, error paths, and side effects — these are exactly what mutation testing rewards.

Once training is complete, upload the model to HuggingFace. The repository name must follow this naming convention:

```
{hf_username}/Qa-{description}-{SS58_hotkey}
```

The prefix can be `Qa`, `qa`, or `QA`. Separators can be dashes `-` or underscores `_`. The last segment must be your miner wallet's full SS58 hotkey address. In our example, alice would name her model:

```
alice/Qa-my_model-5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
```

Other valid formats for the same hotkey:

```
alice/qa_my_model_5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
alice/QA-my_model-5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
```

The repository must contain the actual model weight files. The validator downloads and hashes these files to verify model identity and detect duplicates across the network.

---

## Phase 2: Deploy to Chutes

The deploy script validates the model name, verifies the HuggingFace repo, fetches the revision (commit SHA), deploys to Chutes.ai, and warms up the chute. The on-chain commit is handled by the miner in Phase 3.

**Configure .env.** Set these two variables:

```env
CHUTES_API_KEY=cpk_your_chutes_api_key_here
CHUTE_USER=alice
```

**Run the deployment:**

```bash
python utils/deploy_model.py \
  --model-name "alice/Qa-my_model-5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY"
```

The script runs four steps:

```
[1/4] Validating model name...
[2/4] Verifying HuggingFace...
  HuggingFace: 12 files, has weights
  Revision: a3b1c9f2e8d7654321abcdef0123456789abcdef
[3/4] Deploying to Chutes...
  Chutes deploy OK
[4/4] Warming up...
  Warmup OK
```

At the end, the script prints the values you need for the miner `.env`. To change GPU config, image, concurrency, or other deployment settings, edit `CHUTE_CONFIG` at the top of `utils/deploy_model.py`.

If you already deployed on Chutes manually and just need to warmup, use `--skip-deploy`:

```bash
python utils/deploy_model.py \
  --model-name "alice/Qa-deepseek-coder-v2-5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY" \
  --skip-deploy
```

---

## Phase 3: Run the Miner

With the deployment complete, update the `.env` file with the values from the deploy output:

```env
MODEL_NAME=alice/Qa-deepseek-coder-v2-5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
REVISION=a3b1c9f2e8d7654321abcdef0123456789abcdef
CHUTE_USER=alice
CHUTE_ID=a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

Start the miner:

```bash
python neurons/miner.py \
  --netuid 389 \
  --subtensor.network test \
  --wallet.name miner_wallet \
  --wallet.hotkey default \
  --model-name "alice/Qa-deepseek-coder-v2-5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY" \
  --revision "a3b1c9f2e8d7654321abcdef0123456789abcdef" \
  --axon.port 8091
```

The miner commits JSON data on-chain via `set_reveal_commitment()` and keeps the axon alive. It does not perform inference — the validator calls your model on Chutes directly using the model name. The chute_id is auto-generated from CHUTE_USER + model name if CHUTE_ID is not set. After each evaluation round, the miner receives a ScoreFeedback synapse from the validator with its score, EMA, baseline, required threshold, and rank. Keep the process running so the validator can discover you on the metagraph.

For production, use `pm2` or `systemd` to keep it running persistently:

```bash
pm2 start neurons/miner.py --name qa-miner --interpreter python3 -- \
  --netuid 389 --subtensor.network test \
  --wallet.name miner_wallet --wallet.hotkey default \
  --model-name "alice/Qa-deepseek-coder-v2-5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY" \
  --revision "a3b1c9f2e8d7654321abcdef0123456789abcdef" \
  --axon.port 8091
```

---

## Environment Configuration Reference

### Shared by both roles

```env
NETWORK=test                                              # "test" for testnet, "finney" for mainnet
NETUID=389                                                # Subnet netuid
SUBTENSOR_NETWORK=test                                    # Subtensor network identifier
SUBTENSOR_CHAIN_ENDPOINT=wss://test.chain.opentensor.ai:443

CHUTES_API_KEY=cpk_your_key_here                          # Required for both roles
CHUTES_BASE_URL=https://llm.chutes.ai/v1                 # Chutes API endpoint
```

### Miner-specific

The miner needs its model name, revision, and chute identity (all returned by `deploy_model.py`). The model name serves as both the HuggingFace identifier and the Chutes API model. CHUTE_ID is auto-generated from CHUTE_USER + MODEL_NAME if not set explicitly. The axon port is where the validator discovers the miner on the metagraph.

```env
MODEL_NAME=alice/Qa-deepseek-coder-v2-5Grwva...          # Full HuggingFace model name (= Chutes model)
REVISION=a3b1c9f2e8d7654321abcdef0123456789abcdef         # HF commit SHA from deploy_model.py
CHUTE_USER=alice                                          # Your Chutes.ai username
CHUTE_ID=a1b2c3d4-e5f6-7890-abcd-ef1234567890             # Pre-generated chute ID (auto if CHUTE_USER set)
AXON_PORT=8091                                            # Axon port (validator discovery)
AXON_IP=0.0.0.0                                           # Bind address
AXON_EXTERNAL_PORT=8091                                   # External port (if behind NAT/firewall)
```

### Optional

```env
HF_TOKEN=hf_your_token_here                               # HuggingFace token (avoids rate limits)
```
