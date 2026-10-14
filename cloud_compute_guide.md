
# Cloud Compute Renting for Vast.AI  

> [!info] What this covers
> This is a comprehensive guide and tips on how to use and/or rent a GPU or Cloud Compute. I cannot reply to all of your messages so I created this. This guide will guide you through the setup, renting your model, and using that rented model to train your ML models. Keep in mind that this guide is focused on Vast.AI only.

## 0. Cloud Computes Alternatives

Before you decide to rent or put money into your vast.ai account assess your model first. The Free Tiers from other cloud computes can cover small models and experiments.

|             | Google Colab                   | Kaggle Notebooks             | vast.ai (rest of guide) |
| ----------- | ------------------------------ | ---------------------------- | ----------------------- |
| Free GPU    | T4 16 GB                       | P100 16 GB / T4 ×2           | paid, any GPU           |
| Session     | ~12 h, ~90 min idle disconnect | ~12 h                        | persistent instance     |
| Quota       | dynamic, unpublished           | ~30 h GPU/wk (floats)        | pay per hour            |
| Persistence | Google Drive                   | versioned outputs / datasets | disk (billed)           |

> [!tip] Please Try the Free Tier First!
> Small models and experiments is more than enough in free tiers. Move to vast.ai when you outgrow ~12 hour sessions.

Now to Vast.AI...
### Three ways to work with it

| Path               | Use when                                                |
| ------------------ | ------------------------------------------------------- |
| VS Code Remote-SSH | You want an IDE on the instance (edit, terminal, run)   |
| WSL                | You want a plain Linux terminal on your own machine     |
| Windows PowerShell | You can't or won't use WSL                              |

Renting/creating instances runs on your machine. Training runs on the instance. VS Code Remote-SSH makes the instance side painless.

## 1. Account, credit, API key

1. Sign up at VAST.AI. Use my refferal link https://cloud.vast.ai/?ref_id=716435.
2. Add credit: https://cloud.vast.ai/billing/ (card or crypto). 
3. Create an API key: https://cloud.vast.ai/manage-keys/ then copy it.


> [!warning] API key is like your password
> Anyone with it controls your account and spends your credit. Please do not put your API key anywhere.

## 2. Install the CLI
You can use WSL or Powershell for this.

------------------------
For WSL
WSL:
```bash
curl -fsSL https://vast.ai/install.sh | bash
```
```bash
source ~/.bashrc
```
```bash
vastai --version
```

-------------------------
For Powershell
PowerShell:
```powershell
pip install vastai
```
```powershell
vastai --version
```

---------------------------------------------------------------------

Authenticate:
```bash
vastai set api-key <YOUR_API_KEY>
```
```bash
vastai show user
```

## 3. SSH key

Check for one:
```bash
ls ~/.ssh/id_ed25519.pub
```

If the path shows up. Then you're good.

If there's none, run this:

```bash
ssh-keygen -t ed25519 -C "put-your-gmail-address-here"
```

Register it (WSL):
```bash
vastai create ssh-key "$(cat ~/.ssh/id_ed25519.pub)"
```
Register it (PowerShell):
```powershell
vastai create ssh-key (Get-Content $HOME\.ssh\id_ed25519.pub)
```

## 4. Local machine setup

### 4a. WSL

```powershell
wsl --install -d Ubuntu
```
Then, in Ubuntu:
```bash
sudo apt update
```
```bash
sudo apt install -y git python3 python3-venv python3-pip
```
```bash
git config --global user.name "put-your-username-here"
```
```bash
git config --global user.email "put-your-gmail-address-here"
```

Clone your project:
```bash
git clone <repo-url> 
```
```bash
cd ~/project
```

Local venv:
```bash
python3 -m venv .venv
```
```bash
source .venv/bin/activate
```
```bash
pip install --upgrade pip
```

### 4b. Windows PowerShell

```powershell
winget install --id Git.Git
```
```powershell
winget install --id Python.Python.3.12
```

OpenSSH ships with modern Windows. Verify:
```powershell
ssh -V
```
```powershell
scp -V
```

Clone:
```powershell
git clone <repo-url> "$HOME\project"
```
```powershell
cd "$HOME\project"
```

Local venv:
```powershell
python -m venv .venv
```
```powershell
.\.venv\Scripts\Activate.ps1
```
```powershell
pip install --upgrade pip
```

> [!warning] Two venvs, two machines
> Local venv = your machine. Instance venv = the rented box. Never copy between them.

### 4c. bash → PowerShell

| Task | bash | PowerShell |
|---|---|---|
| download | `curl -fsSL <url>` | `curl.exe -fsSL <url>` |
| subcommand | `$(cmd)` | `$(cmd)` (PS7) / `(cmd)` (PS5) |
| read file | `cat file` | `Get-Content file` |
| env var | `export X=y` | `$env:X="y"` |
| activate venv | `source .venv/bin/activate` | `.\.venv\Scripts\Activate.ps1` |
| chain | `a && b` | `a; b` |
| install CLI | `curl ... \| bash` | `pip install vastai` |

> [!note] In PowerShell, `curl` is an alias for `Invoke-WebRequest` — type `curl.exe` for real curl.

## 5. Choosing an offer

You can choose a model from the Website's UI or in your Powershell or WSL. If you want to just search for it in your Powershell or WSL. You can use this template: 

```bash
vastai search offers "num_gpus=1 gpu_ram>=12 compute_cap>=800 cpu_cores_effective>=4 cpu_ram>=16 reliability>0.98 verified=true rentable=true direct_port_count>=1" -o 'dph_total-'
```

| Filter | Meaning |
|---|---|
| num_gpus | GPUs per instance |
| gpu_ram | VRAM per GPU (GB) — sets batch size |
| compute_cap | Architecture. 800 = Ampere (bf16), 890 = Ada |
| cpu_cores_effective | CPUs you actually get |
| cpu_ram | System RAM (GB) |
| cuda_vers | Host's max CUDA; must fit your torch wheel |
| reliability | Host uptime score |
| verified | Identity-checked host |
| rentable | Available now |
| direct_port_count | Open ports for `--direct` SSH |
| -o 'dph_total-' | Sort by total $/hr, cheapest first |

If you find the model that you want. Get its `OFFER_ID` image and copy it. Put it on the next step so we can now create our instance.
## 6. Create the instance

```bash
vastai create instance <OFFER_ID> --image nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04 --disk 40 --ssh --direct --label ml_training
```
The JSON `new_contract` is your `<INSTANCE_ID>`.

> [!warning] Storage bills while stopped
> Disk size is fixed at creation and billed every second the instance exists so please assess whether you need more than 40GB.

Check status:
```bash
vastai show instances
```
Get the SSH string:
```bash
vastai ssh-url <INSTANCE_ID>
```

## 7. First login + env

```bash
ssh $(vastai ssh-url <INSTANCE_ID>)
```
```powershell
ssh (vastai ssh-url <INSTANCE_ID>)
```

Inside the instance:
```bash
apt-get update
```
```bash
apt-get install -y curl git
```

Clone your repository on the instance and install your dependencies. torch is the big one, so grab the CUDA 12.4 build. Without this, the GPU check below will fail.

```bash
git clone <repo-url>
```
```bash
cd project
```
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```
```bash
source "$HOME/.local/bin/env"
```
```bash
uv venv --python 3.12 --seed .venv
```
```bash
source .venv/bin/activate
```
```bash
uv pip install torch --index-url https://download.pytorch.org/whl/cu124
```
```bash
uv pip install -r requirements.txt
```

Then run this to check the GPU. If it prints `True`, then you're good.
```bash
python -c "import torch; print(torch.cuda.is_available())"
```

## 8. VS Code Remote-SSH

1. Install the **Remote - SSH** extension.
2. Get the connection string:
```bash
vastai ssh-url <INSTANCE_ID>
```
3. Add it as a Host in `~/.ssh/config`, or use Command Palette → Remote-SSH: Connect to Host.
4. Connect, then **Open Folder** on the instance repo.
5. Use the integrated terminal for the venv and training.
6. Monitor your training on the instance — see Section 10.

> [!tip] No quoting to learn
> Once connected, everything (activate venv, run `train.py`, monitor) happens in the instance's integrated terminal — no WSL/PowerShell juggling.

## 9. Pre-flight checklist

Before y'all start training, run through these checks. If any of them fail, fix it first.

Check the GPU is there:
```bash
python -c "import torch; print(torch.cuda.is_available())"
```
This should print `True`.

Check torch and which CUDA build:
```bash
python -c "import torch; print(torch.__version__, torch.version.cuda)"
```

Check you're in the virtual environment:
```bash
which python
```
This should print a path ending in `project/.venv/bin/python`, not the system Python.

Check your repository is cloned:
```bash
ls ~/project
```

Check your dependencies are installed:
```bash
uv pip list
```

## 10. Run training

```bash
tmux new -s train
```
```bash
cd ~/project
```
```bash
source .venv/bin/activate
```
```bash
python train.py --epochs 50
```

Monitor:
```bash
nvidia-smi
```
```bash
df -h
```
```bash
tail -f logs/train.log
```

## 11. Retrieve results

```bash
git add .
```
```bash
git commit -m "training run"
```
```bash
git push
```

Or copy back:
```bash
scp -P <PORT> -r root@<HOST>:~/project/checkpoints ./checkpoints
```

> [!note] Bandwidth is billed
> Download only what you need; avoid pulling large media or logs.

## 12. Stop vs destroy

```bash
vastai stop instance <ID>
```
```bash
vastai destroy instance <ID>
```

| Command | GPU billing | Storage billing | Data |
|---|---|---|---|
| stop | ends | **continues** | kept |
| destroy | ends | ends | deleted |

> [!warning] Destroy when done
> `stop` ends GPU charges but storage keeps billing every second. To fully stop paying, `destroy`.

## 13. Troubleshooting

| Symptom | Fix |
|---|---|
| `cuda.is_available()` False | Driver too old for your torch wheel; filter `cuda_vers` or reinstall a matching wheel |
| "No matching distribution" for numpy/torch | Instance Python too old; `uv venv --python 3.12` or a newer image |
| Clone asks for a password | Private repo over HTTPS; use the SSH URL with a registered key |
| SSH Permission denied | Register the key before creating the instance |
| Disk full | Disk is fixed; destroy and recreate larger; delete checkpoints |
| Run killed after disconnect | Not in `tmux`/`screen` |
| Instance `exited`/`offline` | Won't recover; destroy and rent another |
| Spot preempted | Resume from checkpoints; use on-demand next time |
