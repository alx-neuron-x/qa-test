# QA-Subnet: Automated Unit Test Generation on Bittensor

QA-Subnet is a Bittensor subnet dedicated to advancing the state of AI-generated software testing. Miners compete by fine-tuning large language models that write high-quality Python unit tests, while validators evaluate those tests through mutation testing inside secure sandboxes. The result is a decentralized marketplace where models are continuously refined to produce tests that truly verify code correctness, not just pass superficially.

The subnet operates on **netuid 389** (testnet). Validators call each miner's model directly through Chutes.ai, removing the need for trust in miner-side inference.

---

## Why QA-Subnet Matters

Software testing is one of the most time-consuming phases of the development cycle. Developers often write tests that cover the obvious cases but miss subtle edge conditions, off-by-one errors, and unhandled branches. Existing AI code assistants generate tests, but there is no systematic way to measure whether those tests actually catch real bugs.

QA-Subnet solves this by using **mutation testing** as the ground truth for test quality. Instead of asking "do these tests pass?", the subnet asks "if we introduce bugs into the code, do these tests catch it?". This creates a quantifiable, objective metric for test effectiveness that drives continuous improvement.

For the Bittensor ecosystem, QA-Subnet contributes a specialized intelligence that complements code generation subnets. While other subnets focus on writing code, QA-Subnet focuses on verifying it. The fine-tuned models that emerge from this competitive process can be used by any developer or organization to automatically generate high-coverage test suites, reducing bugs in production and accelerating software delivery.

For AI research more broadly, the subnet produces a growing dataset of (code, test, mutation-score) triples that captures what makes a test effective at catching real defects. This dataset, built from perfect-score evaluations, can fuel the next generation of code-aware AI models.

---

## How It Works

QA-Subnet has two roles: **miners** and **validators**.

**Miners** are developers or teams who fine-tune a language model to be excellent at writing Python unit tests. Once the model is ready, the miner uploads it to HuggingFace, deploys it on Chutes.ai so it can serve inference requests, and registers it on the Bittensor blockchain. After registration, the miner's job is done — the process stays running to maintain network presence, but the miner itself does not perform any inference. This is intentional: by having the validator control all inference calls, the network guarantees that the tests were genuinely produced by the registered model.

**Validators** orchestrate the entire evaluation cycle. In each round, the validator generates a Python code challenge (drawn from a library of over 60 templates covering algorithms, data structures, string manipulation, and more). It then selects a batch of miners and verifies that each miner's model is legitimate through a thorough 10-step verification process. For every verified miner, the validator calls that miner's model on Chutes.ai, asking it to produce unit tests for the challenge. The generated tests are first scanned for prohibited patterns (such as code introspection that could game the system), then executed inside a Docker sandbox with no network access, limited CPU, and limited memory. Inside the sandbox, **mutation testing** runs: the code under test is systematically modified (operators swapped, conditions negated, statements deleted) and the tests are re-executed against each mutation. If the tests catch the mutation (i.e., they fail when the bug is introduced), that mutant is considered "killed". The miner's score is the ratio of killed mutants to total mutants.

Scores are smoothed using an exponential moving average (EMA) across rounds, and rewards are distributed to the top-performing miners. After each round, the validator sends real-time feedback to every evaluated miner via a ScoreFeedback synapse, so miners can monitor their score, EMA, baseline, required threshold, and rank without external tools. The entire cycle — challenge generation, model verification, inference, sandbox execution, scoring, feedback, and weight setting — happens automatically every round without human intervention.

---

## Reward System

The reward system is designed to be fair, resistant to gaming, and to encourage genuine improvement over time.

### Scoring

Each evaluation produces two independent metrics:

**Mutation Score** (0.0 to 1.0) measures how many code mutations the generated tests detected. The source code is systematically modified (operators swapped, conditions negated, statements deleted) and the tests are re-run against each mutation. If the tests catch the mutation (i.e., they fail when the bug is introduced), that mutant is "killed". The score is `killed_mutants / total_mutants`.

**Quality Ratio** (0.0 to 1.0) measures how many of the submitted tests actually pass on the unmodified code. If a model generates 9 tests but 1 has a bug and fails, quality is `8 / 9 = 0.8889`. This penalizes models that produce broken tests.

The final score is the product of both:

```
final_score = mutation_score * quality_ratio
```

For example:

| Scenario | Tests generated | Pass | Quality | Mutants killed | Score | Final |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Good model | 10 | 10 | 1.00 | 73% | 0.73 | **0.73** |
| 1 broken test | 9 | 8 | 0.89 | 74% | 0.74 | **0.66** |
| Bad model | 10 | 5 | 0.50 | 80% | 0.80 | **0.40** |

The bad model kills more mutants (0.80) but half its tests fail, so its final score (0.40) is much worse. It is not enough to kill mutants — the tests must also be valid.

The mutation testing pipeline combines traditional mutations from mutmut (operator swaps, constant changes, boolean flips) with a custom AST-based engine that applies six harder mutation types: statement deletion, off-by-one modifications, full condition negation, return-value nullification, else/elif removal, and statement reordering. This combination ensures that only tests with deep coverage of branches, boundaries, side effects, and execution order achieve high scores.

### EMA Smoothing

Raw scores are not used directly for weight setting. Instead, each miner's score is blended into a running **exponential moving average** with alpha = 0.02. This means each new score contributes only 2% to the moving average, requiring approximately 100 rounds for ~87% convergence. A single outstanding or poor round has minimal long-term impact — consistent performance is what matters.

$$\text{EMA}_t = \alpha \cdot \text{score}_t + (1 - \alpha) \cdot \text{EMA}_{t-1} \qquad \alpha = 0.02$$

The first time a miner is evaluated, its EMA is seeded with the actual score ($\text{EMA}_0 = \text{score}_0$) rather than starting from zero, so the baseline and rankings reflect real performance from the first round. If a miner goes offline, its EMA decays by 5% per round ($\text{EMA}_t = 0.95 \cdot \text{EMA}_{t-1}$). After 5 consecutive offline rounds, the EMA is reset to zero so the miner starts fresh when it returns. Miners that time out three consecutive times receive a -0.1 EMA penalty.

### Improvement Barrier and Top-3 Distribution

UID 0 acts as the **baseline reference**. It runs the default model and sets the minimum performance bar. The dynamic improvement barrier adjusts automatically based on where the baseline stands:

$$p = 10.0 - 9.9 \times B_0$$

$$S_{req} = B_0 \times \left(1 + \frac{p}{100}\right)$$

Where $B_0$ is the baseline (UID 0's EMA), $p$ is the required improvement percentage, and $S_{req}$ is the minimum score to earn rewards. When the baseline is low there is plenty of room for improvement, so the bar is high (up to 10%). As the baseline approaches 1.0 and real gains become harder, the required improvement shrinks to 0.1%. New models (detected by a change in model hash) must exceed this threshold; existing models with the same hash are exempt.

```
Score
1.00 ┤
     │                                          ╭───  Miner EMA
0.90 ┤                                     ╭────╯
     │                                ╭────╯
0.80 ┤                           ╭────╯
     │                     ╭─────╯
0.70 ┤                ╭────╯
     │           ╭────╯
0.60 ┤      ╭────╯
     │  ╭───╯
0.50 ┤──╯
     │                           ···················  Required Score
0.45 ┤                   ·······
     │           ········  ─────────────────────────  Baseline (UID 0)
0.35 ┤   ········  ─────────
     │···  ─────────
0.25 ┤─────
     │
0.00 ┤──────────────────────────────────────────────
     └──┬──────┬──────┬──────┬──────┬──────┬──────┬─→
        0     20     40     60     80    100    120
                              Rounds
```

**Reading the chart**: The miner starts with a low EMA that converges upward as consistent scores accumulate ($\alpha = 0.02$, ~100 rounds for 87% convergence). The baseline (UID 0) also rises over time as better models push the reference score higher. The required score tracks just above the baseline by the dynamic improvement percentage — the gap narrows as the baseline climbs. The miner only starts earning rewards once its EMA crosses above the required score line.

| Baseline | Improvement % | Required Score | Gap |
|:-:|:-:|:-:|:-:|
| 0.10 | 9.01% | 0.1090 | +0.009 |
| 0.30 | 7.03% | 0.3211 | +0.021 |
| 0.50 | 5.05% | 0.5253 | +0.025 |
| 0.70 | 3.07% | 0.7215 | +0.021 |
| 0.90 | 1.09% | 0.9098 | +0.010 |

Only the **top 3 miners** that clear the barrier receive rewards, split as 60% for first place, 30% for second, and 10% for third. Any unclaimed reward slots are absorbed by UID 0. For example, if only one miner surpasses the baseline, that miner receives 60% and UID 0 receives the remaining 40%. If no miner surpasses the baseline, UID 0 receives 100%. This ensures that reward is never distributed to underperforming models and that there is always a strong incentive to beat the reference.

### Offline Decay

Miners that stop serving are tracked round by round. Their EMA decays by 5% per round while offline. After 5 consecutive offline rounds, the EMA is fully reset to zero. After 10 consecutive rounds without presence, the miner is removed from the selection pool entirely. When a miner comes back online after an extended absence, it starts fresh with no historical score, competing on current performance alone. This keeps the network clean and rewards only active participants.

---

## Hardware Requirements

### Validator

The validator needs to run Docker containers for mutation testing and maintain a persistent connection to the Bittensor blockchain. The recommended setup is a VPS or dedicated server with at least 4 CPU cores, 8 GB of RAM, 50 GB of SSD storage, and a stable internet connection. The minimum viable configuration is 16 CPU cores and 128 GB of RAM, though evaluation rounds will be slower. Docker must be installed and the user must have permission to run containers. Ubuntu 22.04+ is the recommended operating system, though any Linux distribution with Docker support will work.

### Miner

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

The `.env` file controls behavior for both miners and validators. The sections below explain what each role needs to configure.

---

## Running a Validator

### Prerequisites

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

### Configuration

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

The remaining validator settings (`DOCKER_TIMEOUT`, `INFERENCE_TIMEOUT`) have sensible defaults and typically do not need changes. EMA smoothing (alpha = 0.02) and batch size (up to 10 miners per round) are hardcoded in the validator. See the Environment Configuration Reference below for the full list.

### Starting the Validator

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

---

## Running a Miner

Mining on QA-Subnet is a three-phase process: fine-tune a model, deploy it to Chutes, and run the miner process. This section walks through a complete example using a hypothetical miner called `alice` with hotkey `5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY`.

### Phase 1: Fine-Tune Your Model

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

### Phase 2: Deploy to Chutes

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

### Phase 3: Run the Miner

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

Both the miner and validator read from the same `.env` file, but each role uses different variables. The file is organized in sections so you only need to fill in the parts relevant to your role.

### Shared by both roles

Every node on the subnet needs to know which network and netuid to connect to, and both the validator and miner need a Chutes API key (the validator uses it to call miner models for inference; the miner uses it during deployment).

```env
NETWORK=test                                              # "test" for testnet, "finney" for mainnet
NETUID=389                                                # Subnet netuid
SUBTENSOR_NETWORK=test                                    # Subtensor network identifier
SUBTENSOR_CHAIN_ENDPOINT=wss://test.chain.opentensor.ai:443

CHUTES_API_KEY=cpk_your_key_here                          # Required for both roles
CHUTES_BASE_URL=https://llm.chutes.ai/v1                 # Chutes API endpoint
```

### Validator-specific

The validator needs Docker for mutation testing. UID 0 must be running as a real miner — the validator reads its model from on-chain data automatically.

```env
DOCKER_TIMEOUT=600                                        # Max seconds per sandbox run
MUTATION_TESTING_ENABLED=true                             # Enable mutation testing
INFERENCE_TIMEOUT=600                                     # Max seconds for Chutes API calls per miner
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

These variables are not required but can be useful for debugging, monitoring, or avoiding rate limits.

```env
LOG_LEVEL=INFO                                            # DEBUG, INFO, WARNING, ERROR
ENABLE_WANDB=false                                        # Weights & Biases dashboard
WANDB_API_KEY=                                            # W&B key (wandb.ai/authorize)
HF_TOKEN=hf_your_token_here                               # HuggingFace token (avoids rate limits)
MAX_RETRIES=3                                             # API call retry count
RETRY_DELAY=2                                             # Seconds between retries
```

---

## Integrity and Security

QA-Subnet employs multiple layers of protection to ensure fair competition and prevent gaming.

All inference is controlled by the validator, which calls each miner's model through Chutes.ai directly. Miners never handle their own inference requests, eliminating the possibility of substituting a different model or modifying outputs in transit. The 10-step model verification process confirms that each miner's on-chain registration, HuggingFace repository, and Chutes deployment are consistent and legitimate before any evaluation takes place. Each deployment is pinned to a specific HuggingFace revision (commit SHA), which the validator uses to verify and hash the model at the exact version that was deployed. This prevents miners from swapping model weights on HuggingFace after registration.

Generated test code is scanned for prohibited patterns before execution. Any tests that attempt to inspect the source code through introspection (examining bytecode, parsing the AST, using reflection) are automatically rejected with a zero score.

Mutation testing itself runs inside a hardened Docker sandbox with no network access, restricted CPU and memory, and strict time limits. This prevents any form of external communication or resource abuse during evaluation.

The reward system uses model hashing and duplicate detection to prevent Sybil attacks where the same model is registered under multiple identities. The dynamic improvement barrier ensures that new registrations must demonstrably outperform the baseline, and the top-3 distribution ensures rewards are concentrated on models that genuinely advance test quality rather than being spread thinly across marginal participants.

---

## License

MIT License. See [LICENSE](LICENSE) for details.
