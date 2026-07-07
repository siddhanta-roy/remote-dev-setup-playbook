# Remote Python & ML Development — A Portable Playbook

*A step-by-step, first-principles guide to setting up a professional remote development environment for scientific Python, machine learning, and Physics-Informed Neural Networks — on any Linux server, in any organization.*

**Author's note:** This playbook was distilled from a real setup session on a corporate engineering cluster. Names, hostnames, and proxy URLs are placeholders you will substitute for your environment. The *process* is universal; the *values* are not.

---

## Table of Contents

1. [The Mental Model — What Are We Actually Building?](#chapter-1)
2. [SSH and VS Code Remote Development](#chapter-2)
3. [Bash Fundamentals You Cannot Skip](#chapter-3)
4. [The Proxy Layer — Making Networks Work Behind Firewalls](#chapter-4)
5. [Filesystem Hygiene — Where to Put Things and Why](#chapter-5)
6. [Python Virtual Environments — Isolation as a Discipline](#chapter-6)
7. [The Scientific Python & ML Stack — First Principles](#chapter-7)
8. [VS Code Extensions on the Remote Side](#chapter-8)
9. [Git — Version Control from Day One](#chapter-9)
10. [SSH Keys — Killing the Password + MFA Dance](#chapter-10)
11. [The Daily Reconnect Workflow](#chapter-11)
12. [Troubleshooting Playbook](#chapter-12)

**Appendices**

- A. [Command Reference Cheat Sheet](#appendix-a)
- B. [Package Quick-Reference Table](#appendix-b)
- C. [Config File Locations Table](#appendix-c)

---

<a name="chapter-1"></a>

## Chapter 1 — The Mental Model: What Are We Actually Building?

Before typing a single command, you need a clear picture of what "remote development" means. Get this right and everything else falls into place.

### The three-machine model

In modern engineering, your code rarely runs where you type it. There are almost always three logical machines involved:

1. **Your laptop** — the *editor* machine. Runs VS Code, your browser, your Slack. Comfortable, familiar, but usually underpowered for real work and not connected to your organization's internal data.
2. **The remote host** — the *compute* machine. A Linux server (often called an "engineering cluster node", "HPC node", or "dev server"). Where your code actually runs, where the data lives, where the licensed software (Cadence, MATLAB, Ansys) is installed, and where any GPUs live.
3. **The proxy / gateway** — the *network policeman*. Corporate networks almost always sit behind proxies and firewalls. Anything that needs the public internet (like pip installing PyTorch, or VS Code fetching an extension) has to route through the proxy.

**Why this matters:** every problem you will hit during setup is at one of the boundaries between these three. Understanding which boundary is broken is 80% of debugging.

### What VS Code Remote-SSH does

VS Code Remote-SSH gives you a very clever illusion: the editor appears to run on your laptop, but every file it reads, every terminal it opens, every Python it runs, actually happens on the remote host. It works by:

1. Opening an SSH connection from your laptop to the remote.
2. Automatically downloading and installing a lightweight companion program ("VS Code Server") on the remote.
3. Streaming just the UI events (keystrokes, mouse clicks) back and forth over the SSH tunnel.

The result: you get local-editor responsiveness with remote-machine power. This is the modern professional workflow, and it beats "SSH into a terminal and use vim" for most engineering work.

### What "installed on remote" means

You will see this phrase constantly, and it trips up beginners. A VS Code extension can live in one of two places:

- **Local** — installed on your laptop, useful only for things your laptop does (like the Remote-SSH extension itself).
- **Remote** — installed on the Linux host via the VS Code Server, useful for things the remote does (Python, Jupyter, linting your Linux-side code).

**Rule of thumb:** if the extension does something with your *files* or your *language* (Python, C++, TypeScript), it must be installed on **remote**. If it does something with the *editor UI* (themes, keybindings, Remote-SSH itself), it can be **local**.

---

<a name="chapter-2"></a>

## Chapter 2 — SSH and VS Code Remote Development

### What SSH is, at a molecular level

SSH ("Secure Shell") is a protocol for opening an encrypted, authenticated interactive session on a remote computer. Since 1995 it has replaced telnet, rlogin, and every other insecure remote-access tool. Every serious server on Earth speaks SSH on TCP port 22.

An SSH connection has three phases:

1. **TCP handshake** — your laptop reaches the server's port 22.
2. **Key exchange** — laptop and server agree on encryption keys for the session (Diffie-Hellman under the hood).
3. **Authentication** — the server verifies you are who you say you are, via password, key, or multi-factor.

Once these are done, everything you type is encrypted; nobody on the network can read it. Even your organization's IT team cannot easily see what you type inside an SSH session (though they log connection metadata).

### Installing and setting up VS Code Remote-SSH

**Prerequisites on your laptop:**

- VS Code installed (get it from `code.visualstudio.com`).
- An SSH client. Windows 10+ has OpenSSH built in; macOS and Linux do too.

**Step 1 — Install the Remote-SSH extension in VS Code**

Open VS Code. Press `Ctrl+Shift+X` (or `Cmd+Shift+X` on macOS) to open the Extensions view. Search for **"Remote - SSH"** by Microsoft. Click Install. This is one of the few extensions that must be installed *locally* on your laptop.

**Step 2 — Connect to a remote host**

Press `Ctrl+Shift+P` to open the command palette. Type **"Remote-SSH: Connect to Host"** and press Enter. It will ask for a hostname; type it in the form `username@hostname.example.com` — for example `sroy5@fc8engele01.gfoundries.com`. Press Enter.

VS Code will open a new window. In the bottom-left corner you'll see a small colored pill that says something like `SSH: hostname`. This is the visual confirmation that you are now editing "on" the remote machine.

On the first connection, VS Code will download the VS Code Server binary onto the remote (about 100 MB). This takes 30–120 seconds. Subsequent connections skip this and take ~15 seconds.

**Step 3 — Authenticate**

Depending on the server's policy, you may be prompted for:

- A **password** — usually your corporate/AD password.
- A **verification code** — a 6-digit code from Google Authenticator, DUO, RSA token, or similar. This is called MFA (multi-factor authentication) and is required by most enterprise servers.
- A **private key** — if you set up key authentication (see Chapter 10).

Type each prompt carefully. If MFA rotates every 30 seconds, watch the countdown and start typing when there are at least 15 seconds remaining, or the code will expire mid-entry and you'll get "Permission denied".

### The green pill checklist

After a successful connect you should see:

- **Bottom-left**: green pill reading `SSH: hostname`.
- **Explorer pane**: prompts you to Open Folder.
- **Title bar**: shows `[SSH: hostname]` next to the workspace name.

If any of these are missing or the pill is red, you're not really connected. Do not proceed; troubleshoot first (see Chapter 12).

---

<a name="chapter-3"></a>

## Chapter 3 — Bash Fundamentals You Cannot Skip

Once you're connected, you'll interact with the remote host through a **shell**. On Linux, the two shells you'll see most often are `bash` and `tcsh`. Bash is by far the more common in modern setups, and every Python/ML tutorial on the internet assumes bash syntax. **If your default shell is anything else (tcsh, csh, zsh), your first action should be to make bash your default in VS Code.**

### How to make bash the default in VS Code's integrated terminal

Press `Ctrl+Shift+P` → type **"Terminal: Select Default Profile"** → pick **bash**. New terminals will now open in bash, regardless of your login shell.

### The ten commands you cannot live without

Memorize these. Anything else can be looked up.

| Command | What it does | Example |
|---|---|---|
| `pwd` | Print working directory (where am I?) | `pwd` |
| `ls` | List files. `ls -la` shows hidden and details. | `ls -la` |
| `cd` | Change directory. `cd -` returns to previous. | `cd ~/projects` |
| `mkdir` | Make directory. `-p` creates parents as needed. | `mkdir -p src/utils` |
| `cp` | Copy files. `-r` for directories. | `cp -r src backup` |
| `mv` | Move or rename. | `mv old.py new.py` |
| `rm` | Remove. `-r` recursive, `-f` force. **DANGEROUS.** | `rm file.txt` |
| `cat` | Print file contents. Useful for small files. | `cat requirements.txt` |
| `grep` | Search for text in files. | `grep "torch" *.py` |
| `less` | View long files a page at a time. `q` to quit. | `less log.txt` |

### The `~` (tilde) shortcut

`~` expands to your home directory. On Linux servers this is usually `/home/<yourusername>`. So `~/projects` means "the `projects` folder inside my home". You should use `~` almost every time you refer to your home — it's shorter and portable across users and machines.

### The `-p` flag on `mkdir`

`mkdir -p a/b/c` creates all three nested directories at once, and won't error if some already exist. Without `-p`, you'd have to run `mkdir a`, `mkdir a/b`, `mkdir a/b/c` in three separate commands, and any of them would fail if that directory already existed. **Rule:** always use `mkdir -p`, always. It's the "safe and forgiving" version.

### What the `>` and `>>` do

- `command > file` — run the command and write its output to `file`, overwriting any existing content.
- `command >> file` — same but *append* to `file` if it already exists.

Example: `pip freeze > requirements.txt` writes the list of installed Python packages into `requirements.txt`.

### Getting out of a stuck prompt

If you accidentally type an unclosed quote or a heredoc without terminator, bash will change the prompt from `$` to `>` and wait forever. **Press `Ctrl+C`** to abort and get your prompt back. No damage is done — the incomplete command is discarded.

---

<a name="chapter-4"></a>

## Chapter 4 — The Proxy Layer: Making Networks Work Behind Firewalls

Almost every corporate network sits behind a **forward proxy**: a server that mediates all outbound HTTP/HTTPS traffic. Without knowing about it, none of your commands can reach the internet.

### What a proxy is

Instead of your machine talking directly to `pypi.org` on port 443, you send the request to a proxy server (e.g., `proxy.company.com:8080`), and the proxy fetches the content on your behalf and returns it. This lets IT log, filter, and secure all outbound traffic centrally.

**You have to tell every network-using tool where the proxy lives, separately.** There is no single "system-wide proxy" that applies to everything.

### The five places you need to set the proxy

Here is the checklist. For each layer, we set the proxy so that layer can reach the outside world.

**1. VS Code — for the Marketplace**

Open Settings (`Ctrl+,`) → search "http.proxy" → set the value. Or edit `settings.json` directly:

```json
{
  "http.proxy": "http://proxy.company.com:8080",
  "http.proxyStrictSSL": false,
  "http.proxySupport": "on"
}
```

**Note:** VS Code fetches extensions from your *laptop's* network, so this proxy must be reachable from your laptop, not from the remote host. On a corporate laptop the proxy is usually already available; you may not need to set this at all.

**2. Bash — for `curl`, `wget`, and general shell commands**

Append to `~/.bashrc` on the remote:

```bash
export http_proxy="http://proxy.company.com:8080"
export https_proxy="$http_proxy"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$http_proxy"
export no_proxy="localhost,127.0.0.1,.company.com,.internal"
export NO_PROXY="$no_proxy"
```

Then run `source ~/.bashrc` to apply immediately. Both uppercase and lowercase forms are needed because different tools respect different conventions.

**3. pip — for installing Python packages**

Bash environment variables are *not always* respected by pip (depends on how Python was compiled). The reliable way is:

```bash
pip config set global.proxy http://proxy.company.com:8080
pip config set global.trusted-host "pypi.org files.pythonhosted.org download.pytorch.org"
```

This writes to `~/.config/pip/pip.conf` and is persistent across sessions.

**4. Git — for cloning and pushing repositories**

```bash
git config --global http.proxy  http://proxy.company.com:8080
git config --global https.proxy http://proxy.company.com:8080
```

Stored in `~/.gitconfig`.

**5. Any other tool** — apt, yum, docker, conda, npm, cargo — all have their own proxy configs. Look them up when you install them.

### Discovering the correct proxy URL

If you're at a new organization and don't know the proxy address:

- **Ask IT** — the fastest and most reliable path.
- **Check `/etc/environment` and `/etc/profile.d/*.sh`** on the remote — sometimes it's baked in.
- **Look at `/etc/yum.conf` or `/etc/dnf/dnf.conf`** — package managers often have it.
- **Look at your browser's proxy settings** — on your laptop, the same URL usually works from a corporate Linux host.
- **Try common port scan** — some organizations run proxies on 80, 8080, 3128, 8000. A quick `curl -x http://candidate:PORT https://example.com` will succeed with 200 if it's the right one.

### Why "trusted-host" for pip

Corporate proxies often perform **SSL inspection** — they decrypt your HTTPS traffic, look at it, and re-encrypt it with an internal certificate. Python doesn't trust that internal certificate by default and refuses the connection. Adding hosts to `trusted-host` tells pip to skip certificate verification for those specific domains. This is safe for well-known public repos like `pypi.org`; do not add arbitrary hosts.

---

<a name="chapter-5"></a>

## Chapter 5 — Filesystem Hygiene: Where to Put Things and Why

New engineers often make the same mistake: they put all their work in `~` (home directory) because that's where they land when they log in. This works until it doesn't — usually at the worst possible moment.

### Home is small and precious

On shared engineering servers, your home directory typically has a **quota** — a hard limit of 5 to 20 GB. Home is backed up nightly and often has snapshot support (great for recovering from `rm -rf` accidents). Because of these guarantees, IT keeps it small.

**Rule of thumb:** home is for configs, scripts, notes, dotfiles, and small things. Not for datasets, virtual environments (which grow to 1+ GB), model checkpoints, or simulation dumps.

### Shared project space is where real work lives

Most engineering organizations have a shared filesystem for team work. Names vary:

- `/proj/<teamname>/` — common in semiconductor and HPC shops
- `/work/`, `/workspace/`, `/scratch/`, `/data/` — common in universities
- `/home/shared/`, `/nfs/team/` — small teams

These areas are much larger (terabytes), often not personally-quotaed, and shared with your team. **Ask your manager on day one:** "Where should my project files live? Is there a team convention for personal subdirectories?"

### The hybrid layout — best of both worlds

Professional engineers use a hybrid pattern:

- **`~` (home)** — small: bash configs, SSH keys, dotfiles, VS Code settings, personal scripts.
- **`/proj/team/user/`** — big: source code, virtual environments, datasets, model checkpoints, notebooks.
- **Symlink** — `ln -s /proj/team/user/projects ~/projects` — so you can still write `~/projects/foo` and it "just works" while the actual bytes live on the big shared filesystem.

Create it like this:

```bash
mkdir -p /proj/team/users/myuser/projects
ln -s /proj/team/users/myuser/projects ~/projects
ls -la ~/projects   # confirm the symlink
```

### Recognize a good project directory structure

For any Python/ML project:

```
projectname/
├── .venv/              # virtual environment — .gitignored
├── src/                # your Python source code
├── notebooks/          # Jupyter notebooks for exploration
├── data/               # datasets — usually .gitignored
├── models/             # saved model weights — usually .gitignored
├── outputs/            # plots, logs, results — usually .gitignored
├── requirements.txt    # pinned package versions
├── .gitignore          # what NOT to commit
├── RESTART.md          # notes to future-you for reconnecting
└── README.md           # what this project is about
```

Create it with:

```bash
mkdir -p projectname/{src,notebooks,data,models,outputs}
```

The `{a,b,c}` syntax is called **brace expansion** — bash expands it into multiple arguments, so this creates all five subdirectories in one command.

---

<a name="chapter-6"></a>

## Chapter 6 — Python Virtual Environments: Isolation as a Discipline

If you've only ever done `pip install` at your system Python before, this chapter will change how you work.

### The problem

System Python (the one that comes with your OS) has these problems:

- It's shared with the operating system. Some Linux tools depend on specific package versions there; installing over them can break your OS.
- It's often outdated (RHEL 8 ships Python 3.6, released 2016).
- Every project you work on may need different versions of the same package. Project A needs `numpy 1.20`, project B needs `numpy 2.0` — impossible to have both installed system-wide.
- You need `sudo` to install packages, which you rarely have on shared servers.

### The solution: virtual environments

A **virtual environment** (venv) is a self-contained Python installation, isolated inside a directory of your choice. Each project gets its own. When you "activate" a venv, your shell's `PATH` is modified so that `python` and `pip` point to the venv's binaries, and everything you install goes into the venv's own site-packages directory.

**Consequences:**

- Projects are independent. Numpy 1.20 in project A, numpy 2.0 in project B — no conflict.
- No `sudo` needed. Everything lives in your own directory.
- Deleting `.venv/` completely undoes all your installs — clean reset in one command.
- The venv is not backed up by git (you `.gitignore` it), because it's a build artifact — anyone can recreate it from `requirements.txt`.

### Create and activate a venv

```bash
cd /path/to/projectname

# Pick the newest Python available on your system
which python3            # e.g., /tool/pandora64/bin/python3
python3 --version        # e.g., Python 3.12.0

# Create the venv (takes ~5 seconds)
python3 -m venv .venv

# Activate it
source .venv/bin/activate
```

Your prompt now has `(.venv)` at the front. Verify:

```bash
which python             # should point inside .venv/bin
python --version         # should match what you picked
which pip                # should point inside .venv/bin
```

### Choose the right base Python

If your system has multiple Pythons (`/usr/bin/python3.6`, `/tool/whatever/python3.12`), use the newest available. Modern ML libraries drop old Python support quickly:

- **Python 3.6, 3.7** — dead. Do not use.
- **Python 3.8, 3.9** — okay but aging.
- **Python 3.10, 3.11, 3.12** — current sweet spot.
- **Python 3.13+** — some ML packages don't have wheels yet; safer to wait a few months.

### Deactivate

When you're done, type `deactivate` (a function that the venv adds to your shell). Your prompt returns to normal.

**Common misconception:** deactivating does not delete the venv. It just restores your shell. The venv files stay on disk, ready to be reactivated whenever.

### When to make a new venv

- **Every new project** — always. Cheap to create, priceless when you have multiple projects.
- Not per-experiment. Within a project, keep one venv.

---

<a name="chapter-7"></a>

## Chapter 7 — The Scientific Python & ML Stack: First Principles

Here's what to install and — more importantly — **why**. Understanding the "why" makes you a real user rather than a copy-paster.

### The seven-package minimum

For any scientific/ML Python project:

```bash
pip install --upgrade pip
pip install numpy scipy matplotlib pandas scikit-learn jupyterlab ipykernel
pip install torch --index-url https://download.pytorch.org/whl/cpu
```

Let's go through each.

### `numpy` — the foundation of the entire ecosystem

**What:** N-dimensional array data structure and a huge library of fast vectorized operations on them.

**Why it exists:** Python's built-in lists are slow — each element is a full Python object with a type tag and reference count, stored non-contiguously. NumPy arrays are contiguous blocks of raw numbers (like C arrays), operated on by compiled code (BLAS, LAPACK) at near-hardware speed. Matrix multiplication in numpy is 100–1000x faster than pure Python.

**When you use it:** everywhere. Almost every other scientific package (scipy, pandas, sklearn, pytorch, matplotlib) accepts numpy arrays as input. Learn numpy well before anything else.

**First-principles habit:** whenever you find yourself writing a Python for-loop over numbers, ask *"could this be a numpy vectorized operation?"* — the answer is usually yes, and it will be 100x faster.

### `scipy` — numpy's mathematically-rich big sibling

**What:** A library on top of numpy with routines for optimization, integration, interpolation, linear algebra, statistics, signal processing, sparse matrices, ODEs, and more.

**Why it exists:** numpy handles arrays; scipy handles the algorithms scientists actually need on those arrays. Root-finding, curve fitting, FFTs, solving differential equations — all live in scipy.

**When you use it:** whenever you need a specific numerical algorithm. Examples: `scipy.optimize.curve_fit` to fit a model to data, `scipy.integrate.solve_ivp` to integrate an ODE, `scipy.signal.savgol_filter` to smooth noisy measurements.

### `matplotlib` — plotting

**What:** The most widely-used Python plotting library.

**Why it exists:** you cannot understand data without visualizing it. Matplotlib produces publication-quality static plots and interactive figures. It has the same relationship to Python that MATLAB's plotting has to MATLAB.

**When you use it:** every single time you have numbers you want to look at. Every time. Data exploration, debugging, presenting results — all matplotlib.

**Rule of thumb:** if a numerical result isn't in a plot, you don't really understand it yet. Plot everything.

### `pandas` — tabular data

**What:** DataFrames — 2D tables with named columns, mixed types, and a huge query/aggregation/merge API.

**Why it exists:** most real-world data comes as tables (CSV, Excel, SQL results). Pandas is Python's answer to SQL and R's data frames. It handles missing values, dates, and heterogeneous columns gracefully in a way numpy alone doesn't.

**When you use it:** any time you're loading CSV, Excel, or database data. For pure numerical arrays, stick with numpy. For labelled tables with mixed types, pandas.

### `scikit-learn` — classical machine learning

**What:** Consistent, well-designed implementations of dozens of classical ML algorithms — linear/logistic regression, SVMs, random forests, k-means, PCA, cross-validation, preprocessing utilities.

**Why it exists:** for problems where deep learning is overkill (which is most problems), sklearn gives you the standard ML toolbox with a uniform API: every model has `fit`, `predict`, `score`. Great for baselines.

**When you use it:** any structured/tabular data problem. Fitting a compact model, classification, clustering, dimensionality reduction. Also excellent for the *preprocessing* pipeline (scalers, encoders, splitters) even when you use a deep learning model for the final prediction.

### `jupyterlab` — interactive notebooks

**What:** A web-based interactive computing environment. Notebooks mix code, output, plots, and prose in a single document.

**Why it exists:** the fastest way to explore data and iterate on ideas is to run a small piece of code, look at the output, and repeat. Notebooks are optimized for that loop.

**When you use it:** exploratory data analysis, one-off analyses, teaching, presenting narrative results. Not for production code — refactor important logic into `.py` modules once it stabilizes.

**Pro tip:** VS Code renders Jupyter notebooks natively with the Jupyter extension. You don't need a browser.

### `ipykernel` — the glue between your venv and Jupyter

**What:** A small package that registers your virtual environment as a "kernel" — a Python interpreter that Jupyter can attach to.

**Why it exists:** Jupyter itself is language-agnostic. Kernels are how it knows which Python to run your code in. Without registering your venv as a kernel, notebooks would default to a system Python that doesn't have your packages.

**How to register:**

```bash
python -m ipykernel install --user --name projectname --display-name "Python (projectname)"
```

Now "Python (projectname)" appears in Jupyter's kernel picker.

### `torch` (PyTorch) — deep learning

**What:** A tensor library like numpy, but with two superpowers: automatic differentiation and GPU acceleration.

**Why it exists:** modern neural networks need gradients of a scalar loss with respect to millions of parameters. Writing those by hand is impossible. PyTorch builds a computation graph as you evaluate expressions, then walks it backward to compute gradients automatically (`autograd`).

**Why the `--index-url` flag:** the default PyTorch on PyPI bundles CUDA (GPU) support, which is ~2 GB. If you don't have a GPU (most engineering servers don't), the CPU-only wheel at `https://download.pytorch.org/whl/cpu` is much smaller (~200 MB) and equally functional for CPU work.

**When you use it:** any modern neural network — CNNs, transformers, physics-informed nets, generative models. Also useful for high-dimensional optimization with autograd even outside of "AI" contexts.

### Beyond the minimum

Common additions depending on the project:

| Package | For |
|---|---|
| `seaborn` | Prettier statistical plots on top of matplotlib |
| `plotly` | Interactive HTML plots |
| `sympy` | Symbolic math (algebra, calculus by symbols) |
| `statsmodels` | Statistical modeling with a stats-focused API |
| `tqdm` | Progress bars for long loops |
| `numba` | JIT-compile pure numpy code for 100x speedups |
| `xarray` | Labelled N-D arrays (numpy meets pandas) |
| `h5py` | Read/write HDF5 files (common in scientific data) |
| `scikit-image` | Image processing algorithms |
| `optuna` | Hyperparameter tuning |
| `tensorboard` | Visualize training metrics |
| `pytorch-lightning` | High-level wrapper around PyTorch for cleaner training code |

Install as needed. Do not preemptively install everything — a fresh venv is a clean slate.

### Freeze your dependencies

Immediately after your installs stabilize:

```bash
pip freeze > requirements.txt
```

This writes every installed package with its exact version to `requirements.txt`. To recreate the same environment elsewhere:

```bash
pip install -r requirements.txt
```

Commit `requirements.txt` to git. Do not commit `.venv/`.

---

<a name="chapter-8"></a>

## Chapter 8 — VS Code Extensions on the Remote Side

Once you're connected via Remote-SSH, install these extensions **on the remote** (look for the "Install in SSH: hostname" button in the Extensions view, or right-click any local extension and pick that option).

### Essential

- **Python (Microsoft)** — interpreter selection, debugging, linting. Auto-installs Pylance and Debugpy as dependencies.
- **Pylance** — fast, type-aware IntelliSense. Bundled with Python extension.
- **Jupyter (Microsoft)** — native notebook editing inside VS Code.
- **Python Debugger (Microsoft)** — the modern debug adapter (bundled with Python).

### Highly recommended

- **GitLens (GitKraken)** — inline git blame, line history, branch visualization. Once you use it, hard to go back.
- **Even Better TOML** — syntax for `pyproject.toml` and other TOML files.

### For specific workflows

- **Rainbow CSV** — colorize columns in CSV files opened in the editor.
- **Excalidraw** — sketch diagrams inline.
- **Markdown All in One** — better markdown editing, TOC generation.
- **Remote Explorer** — see and manage SSH hosts as a tree.

### Selecting the interpreter after install

Extensions installed, `.venv` created — now tell VS Code which Python to use:

`Ctrl+Shift+P` → **"Python: Select Interpreter"** → pick `./.venv/bin/python` from the list (marked *Recommended*).

The bottom-right status bar will show `Python 3.12.0 ('.venv': venv)`. From now on:

- New terminals auto-activate the venv.
- IntelliSense knows about your installed packages.
- The debugger uses the venv's Python.
- Jupyter notebooks default to the registered `pinn_gan` (or whatever you named it) kernel.

---

<a name="chapter-9"></a>

## Chapter 9 — Git: Version Control from Day One

Version control is not optional. Use it from the first hour of any project.

### The one-time global config

```bash
git config --global user.name  "Your Name"
git config --global user.email "your.email@company.com"
git config --global init.defaultBranch main
```

### Initialize a repo

```bash
cd /path/to/projectname
git init
```

This creates a hidden `.git/` directory that tracks history.

### The `.gitignore` — as important as your code

Tell git what NOT to track. For Python/ML projects, a solid starter:

```
# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/

# Jupyter
.ipynb_checkpoints/

# Data & outputs (usually too big for git)
data/
outputs/
models/*.pt
models/*.pth
*.h5
*.npz

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Logs
*.log
```

### The commit workflow

```bash
git status                    # what's changed?
git add file1.py file2.py     # stage specific files
git add .                     # stage everything (respects .gitignore)
git commit -m "clear message" # snapshot
git log --oneline             # see history
```

### Writing good commit messages

- First line: 50 chars or less, imperative mood ("Add PDE loss", not "Added PDE loss").
- Blank line, then a longer explanation if needed.
- Explain *why*, not *what* — the diff already shows what.

### Pushing to a remote (GitHub, GitLab, Bitbucket)

Once you have a remote hosted repository:

```bash
git remote add origin https://github.com/user/repo.git
git branch -M main
git push -u origin main
```

Subsequent pushes are just `git push`.

### Why commit early and often

Every commit is a save-point. Broke something and can't figure out what? `git diff` shows exactly what changed since the last commit. Truly broken? `git checkout .` reverts everything to the last commit. This safety net gives you the freedom to experiment aggressively.

---

<a name="chapter-10"></a>

## Chapter 10 — SSH Keys: Killing the Password + MFA Dance

Typing your password and a 6-digit MFA code every time you reconnect is:

- Slow (adds 30–60 seconds per connection).
- Error-prone (mistyped codes = "Permission denied").
- Risky (repeated failures can lock your account).
- Unnecessary (SSH keys are strictly more secure than passwords).

Fix it once, benefit forever.

### The concept

An SSH key pair consists of:

- A **private key** — a file on your laptop. Never share it. Never copy it off your machine.
- A **public key** — a matching file. Safe to share and put on servers.

When you connect, the server challenges your laptop to prove it has the private key. Your laptop proves it via cryptographic signature (no key material transmitted). The server verifies with the public key on file, and grants access.

**Note on MFA:** on some enterprise servers, even key auth still requires MFA (a PAM policy decision). But typically keys bypass MFA on internal engineering hosts.

### Step-by-step

**On your laptop, generate a key pair:**

```bash
# Linux/macOS:
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_serverX -C "you@laptop-to-serverX"

# Windows PowerShell:
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_serverX" -C "you@laptop-to-serverX"
```

Press Enter twice for an empty passphrase (fine for internal corporate use), or set a passphrase and use `ssh-agent` to cache it.

**Display the public key:**

```bash
# Linux/macOS:
cat ~/.ssh/id_ed25519_serverX.pub

# Windows PowerShell:
Get-Content "$env:USERPROFILE\.ssh\id_ed25519_serverX.pub"
```

Copy the single-line output.

**Install it on the remote (via your VS Code SSH terminal on the remote):**

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'PASTE_THE_PUBLIC_KEY_HERE' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

**IMPORTANT — check what filename your organization expects:**

- Standard OpenSSH: `~/.ssh/authorized_keys`
- Some older or custom setups: `~/.ssh/authorized_keys2`

If keys don't work with one, try the other. Ask your IT team if unclear.

**Create a local SSH config for shortcuts:**

`~/.ssh/config` (Linux/macOS) or `$env:USERPROFILE\.ssh\config` (Windows):

```
Host serverX
    HostName serverX.company.com
    User yourusername
    IdentityFile ~/.ssh/id_ed25519_serverX
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

Now you can `ssh serverX` from a terminal, and VS Code Remote-SSH will use the key automatically.

### Verify

```bash
ssh serverX hostname
```

If it prints the remote hostname without asking for a password, you're done. If it still asks, run `ssh -v serverX hostname` and look for lines mentioning `publickey` in the verbose output — that tells you what auth methods were attempted and why they failed.

---

<a name="chapter-11"></a>

## Chapter 11 — The Daily Reconnect Workflow

After the initial setup, your daily-start ritual should be short:

1. Open VS Code on your laptop.
2. `Ctrl+Shift+P` → **"Remote-SSH: Connect to Recent"** → pick your host from the top of the list.
3. If keys are set up, no password/MFA. Otherwise, enter them.
4. Wait ~15 seconds for the green pill.
5. **File → Open Recent** → click your project. Explorer loads.
6. Open a terminal (`Ctrl+backtick`). Venv auto-activates.
7. Optional sanity check: `python src/hello.py` to confirm the environment works.

Total time: under a minute. Compare that to redoing the whole setup — which is why we did it once carefully.

### A restart-guide file for your future self

Every project should have a `RESTART.md` at its root:

```
# Reconnecting to <project>

## From laptop
1. Open VS Code.
2. Remote-SSH: Connect to Host -> serverX.company.com
3. File -> Open Folder -> /proj/team/user/projects/projectname

## Terminal (venv auto-activates)
- python src/hello.py

## Environment
- Python 3.12.0
- Torch 2.12.1+cpu
- Kernel: projectname

## Proxy references (in case anything breaks)
- Bash:  ~/.bashrc -> http_proxy=http://proxy.company.com:8080
- pip:   ~/.config/pip/pip.conf -> global.proxy=http://proxy.company.com:8080
- git:   ~/.gitconfig -> http.proxy=http://proxy.company.com:8080
```

Six months from now on a new laptop, this file is what saves you.

---

<a name="chapter-12"></a>

## Chapter 12 — Troubleshooting Playbook

The most common problems, in the order you'll hit them.

### "Remote-SSH: Connect" spins forever

- The proxy setting for VS Code is trying to route SSH through a proxy that can't reach the server. Add `"http.noProxy": ["*.company.com", "localhost"]` to VS Code settings.
- Or remove `http.proxy` entirely if your laptop reaches the server directly.
- Kill VS Code (Task Manager if stuck) and reopen.

### "Permission denied" during authentication

- Wrong password (usually your Windows/AD password, not your Linux password).
- MFA code expired between viewing and typing — wait for a fresh one with >15 seconds left.
- Account locked from too many failures — contact IT to unlock.

### "Failed to fetch" in Extensions view

- Your laptop can't reach the Marketplace. Check `netsh winhttp show proxy` (Windows) or test with `curl https://marketplace.visualstudio.com`.
- Try removing the VS Code `http.proxy` — many corporate laptops reach Marketplace directly.

### `pip install` gives "Name or service not known"

- pip is doing a direct DNS lookup of `pypi.org` instead of using the proxy.
- Run `pip config set global.proxy http://proxy.company.com:8080`.
- Retry.

### `torch` install fails with SSL certificate error

- The proxy is intercepting SSL. Add `--trusted-host download.pytorch.org` to the pip command, or add it to `pip config set global.trusted-host` permanently.

### Terminal shows `>` and won't return to `$`

- Bash is waiting for you to close a heredoc, quote, or backslash-continued line.
- Press `Ctrl+C` to abort. No damage done.

### "Failed to activate venv" / `.venv` doesn't work after `source activate`

- The venv was created with a Python that no longer exists (e.g., you rebuilt an environment module). Recreate: `rm -rf .venv && python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`.

### Jupyter kernel not found in VS Code notebook

- Register: `python -m ipykernel install --user --name projectname --display-name "Python (projectname)"`
- Reload VS Code window.

### Git says "SSL verify error"

- Same story as pip. Corporate proxy intercepts SSL.
- Fix: `git config --global http.sslVerify false` (loose) or install your organization's CA certificate into git (proper).

### VS Code Server won't start on remote

- Free up space in `~/.vscode-server` (it can grow).
- Or nuke it: `rm -rf ~/.vscode-server` — next connect will reinstall fresh.

### Everything is slow

- Check disk usage: `df -h ~ && du -sh ~/.vscode-server`.
- Check load: `uptime` — a shared host might be overloaded by other users.
- Move to a different node if your cluster has options.

---

<a name="appendix-a"></a>

## Appendix A — Command Reference Cheat Sheet

### Filesystem

```
pwd                          # print current directory
ls -la                       # list all files with details
cd ~/projects                # change to home/projects
mkdir -p a/b/c               # create nested dirs
cp -r src backup             # copy directory recursively
mv old new                   # rename or move
rm file                      # delete file
rm -rf dir                   # delete directory (careful!)
du -sh directory             # size of directory
df -h .                      # disk usage of filesystem
```

### Files

```
cat file                     # print file contents
less file                    # page through file (q to quit)
head -20 file                # first 20 lines
tail -20 file                # last 20 lines
tail -f log                  # follow log as it grows
grep pattern file            # search for pattern
grep -r pattern dir          # search recursively
wc -l file                   # count lines
```

### Networking / Debugging

```
curl -I https://url          # HEAD request, see headers
curl -o file https://url     # download to file
curl -x http://proxy url     # via proxy
ping host                    # basic reachability
nslookup host                # DNS lookup
env | grep -i proxy          # show proxy env vars
```

### Python venv

```
python3 -m venv .venv        # create venv
source .venv/bin/activate    # activate
deactivate                   # leave venv
pip install pkg              # install package
pip freeze > requirements.txt  # save state
pip install -r requirements.txt  # restore state
pip list                     # show installed packages
pip show pkg                 # details on one package
```

### Git

```
git init                     # start repo
git status                   # what's changed
git diff                     # show changes
git add file                 # stage file
git commit -m "message"      # commit
git log --oneline            # history
git checkout file            # revert file
git stash                    # temp save changes
git stash pop                # restore stashed changes
git branch                   # list branches
git checkout -b feature      # create + switch
git merge feature            # merge into current
```

### SSH

```
ssh user@host                # connect
ssh -v user@host             # verbose (debug)
ssh -X user@host             # X11 forwarding
scp file user@host:path      # copy to remote
scp user@host:path file      # copy from remote
ssh-keygen -t ed25519 -f key # generate key
ssh-copy-id -i key user@host # install public key
```

---

<a name="appendix-b"></a>

## Appendix B — Package Quick-Reference Table

| Package | One-line description | Import as |
|---|---|---|
| numpy | N-dimensional arrays and vectorized math | `import numpy as np` |
| scipy | Scientific algorithms on numpy arrays | `import scipy` (submodules imported explicitly) |
| matplotlib | Plotting | `import matplotlib.pyplot as plt` |
| pandas | Tabular data (DataFrames) | `import pandas as pd` |
| scikit-learn | Classical machine learning | `from sklearn.model_selection import train_test_split` |
| jupyterlab | Interactive notebook environment | (not imported, launched with `jupyter lab`) |
| ipykernel | Register venv as Jupyter kernel | (not imported, used via CLI) |
| torch | Deep learning tensors + autograd | `import torch` |
| torchvision | Image datasets/models for PyTorch | `import torchvision` |
| seaborn | Statistical plots on matplotlib | `import seaborn as sns` |
| plotly | Interactive HTML plots | `import plotly.express as px` |
| sympy | Symbolic mathematics | `import sympy as sp` |
| statsmodels | Statistical modeling | `import statsmodels.api as sm` |
| tqdm | Progress bars | `from tqdm import tqdm` |
| numba | JIT-compile numpy code | `from numba import jit` |
| h5py | HDF5 file I/O | `import h5py` |
| xarray | Labelled N-D arrays | `import xarray as xr` |

---

<a name="appendix-c"></a>

## Appendix C — Config File Locations Table

| Config | Location on remote | What it holds |
|---|---|---|
| Bash startup | `~/.bashrc` | env vars, aliases, functions |
| Bash login | `~/.bash_profile` or `~/.profile` | login-time setup |
| pip | `~/.config/pip/pip.conf` | proxy, trusted hosts, index URLs |
| git | `~/.gitconfig` | user identity, proxy, aliases |
| SSH client | `~/.ssh/config` | host aliases, keys, options |
| SSH authorized keys | `~/.ssh/authorized_keys` (or `authorized_keys2`) | public keys allowed to log in |
| Jupyter kernels | `~/.local/share/jupyter/kernels/` | registered kernels |
| VS Code Server | `~/.vscode-server/` | remote-side VS Code data |
| VS Code User settings | (on laptop) `%APPDATA%\Code\User\settings.json` | editor preferences |
| VS Code Remote settings | (on remote) `~/.vscode-server/data/Machine/settings.json` | remote-scope preferences |

---

## Closing Thoughts

The setup you just performed is not a one-time chore. It is a **skill**. Every organization you join, every new project you start, every new server you get access to, the same pattern applies. Once, in an afternoon, you learned it deeply enough that reproducing it takes minutes.

Three habits that separate professionals from copy-paste engineers:

1. **Understand what each command does before running it.** Never paste a command from the internet without asking "what does this do to my system?" This guide's "why" sections exist so that even the pasted commands here you understand.

2. **Keep your setup reproducible.** `requirements.txt`, `.gitignore`, `RESTART.md`, dotfiles in a git repo, all of these together let you re-create your environment on any machine in an hour. Without them, you're rebuilding from memory every time.

3. **Fail loudly and locally.** When something breaks, don't panic and start `sudo`-ing things. Look at the error message. Isolate which of the three layers (laptop, network, remote) is failing. Fix the smallest possible thing. This discipline gets you unstuck in minutes instead of hours.

Good luck. Everything you build on top of this foundation will be easier because the foundation is solid.

---


## Learnings from Applying This Playbook

The `learnings/` folder captures milestone-by-milestone reflections from using this playbook on real projects.

- [2026-07-07: PINN Skeleton, CI, and Portfolio](learnings/2026-07-07-pinn-skeleton-and-ci.md)


*End of playbook.*
