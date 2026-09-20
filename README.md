# HPE Private Cloud AI: first notebook

Code for the training "Agentic AI: Building and Shipping Production Agent Systems", delivered on HPE Private Cloud AI (PCAI) through HPE GreenLake.

This folder holds a notebook that proves a notebook server on PCAI can reach a model served by HPE MLIS, then runs a small LangGraph agent on it. It is the code for Lab 0 in Module 1.

## Contents

| File | What it is |
|---|---|
| `pcai_first_notebook.ipynb` | The notebook. Nine steps, run one cell at a time. |
| `README.md` | This file. |

## What the notebook does

1. Prints where the kernel runs (Python version, hostname, working folder).
2. Installs `langchain-openai`, `langgraph` and `requests` with `%pip`.
3. Reads the endpoint URL, model name and token from environment variables, or asks for them with a hidden prompt.
4. Makes one raw `POST` to `<endpoint>/chat/completions` with the token as a Bearer header.
5. Makes the same call through LangChain (`ChatOpenAI` with a custom `base_url`).
6. Optionally calls an embedding deployment at `<endpoint>/embeddings`.
7. Runs a two-path LangGraph graph for IT incidents: `classify`, then `escalate` for high severity or `resolve` otherwise. The routing decision is plain code on a value in the state.
8. Saves the results to `pcai_first_run_<date>_<time>.json` in the working folder. The token is never written.
9. Wrap-up and a short troubleshooting table.

## Run it on PCAI

1. Sign in to HPE AI Essentials and open **Notebooks**.
2. Start your notebook server (or create one with JupyterLab and an image that has Python 3.10 or later), then connect.
3. Upload `pcai_first_notebook.ipynb` through the JupyterLab file browser, or `git clone` this repository from a notebook terminal if the network allows it.
4. From **MLIS**, copy the deployment's endpoint URL and model name, and create a deployment token.
5. Open the notebook, choose the **Python 3** kernel, and run each cell with `Shift + Enter`. Enter the values when prompted.
6. When finished, stop the notebook server from the Notebook Servers screen.

Menu names can differ between AI Essentials releases. Compare them with your screen.

## Settings

Each setting is read from an environment variable if that variable exists. Otherwise the notebook prompts for it.

| Variable | Required | Meaning |
|---|---|---|
| `MLIS_LLM_BASE_URL` | Yes | Endpoint of the LLM deployment. A trailing `/chat/completions` is trimmed. |
| `MLIS_LLM_MODEL` | Yes | Model name served by that deployment. |
| `MLIS_DEPLOY_TOKEN` | Yes | MLIS deployment token, sent as `Authorization: Bearer <token>`. |
| `MLIS_EMB_BASE_URL` | No | Endpoint of an embedding deployment. Set it to an empty value to skip Step 6. |
| `MLIS_EMB_MODEL` | No | Embedding model name. |
| `MLIS_EMB_TOKEN` | No | Embedding token. Empty means reuse the LLM token. |

## Run it headless

Useful for a health check before a session. Run in a terminal where `jupyter nbconvert` is installed.

```bash
export MLIS_LLM_BASE_URL="<endpoint URL>"
export MLIS_LLM_MODEL="<model name>"
read -s -p "Token: " MLIS_DEPLOY_TOKEN; export MLIS_DEPLOY_TOKEN; echo
export MLIS_EMB_BASE_URL="" MLIS_EMB_MODEL="" MLIS_EMB_TOKEN=""

jupyter nbconvert --to notebook --execute pcai_first_notebook.ipynb \
    --output pcai_first_notebook_run.ipynb
```

The executed copy contains outputs such as endpoint addresses. Do not commit it.

## Run it outside PCAI

Any OpenAI-compatible chat endpoint works.

```bash
python -m venv .venv
source .venv/bin/activate
pip install jupyterlab langchain-openai langgraph requests
export MLIS_LLM_BASE_URL="<endpoint URL>" MLIS_LLM_MODEL="<model name>" MLIS_DEPLOY_TOKEN="<token>"
jupyter lab
```

## Troubleshooting

| What you see | Likely cause | What to do |
|---|---|---|
| `HTTP 401` or `403` | Wrong, expired or wrong-deployment token | Create or copy the deployment token again. |
| `HTTP 404` | Wrong URL path or model name | Add or remove `/v1` at the end of the endpoint. Copy the model name exactly. |
| `HTTP 502`, `503` or a timeout | Deployment starting, scaled to zero, or busy | Check its status in MLIS, wait and retry. |
| SSL or certificate error | Internal certificate authority not trusted by Python | Get the CA file from your administrator and set `SSL_CERT_FILE` and `REQUESTS_CA_BUNDLE` before Step 4. |
| `ModuleNotFoundError` | Kernel restarted, so `%pip` installs in the base environment were removed | Re-run Step 2, restart the kernel, continue. |
| Answer contains `<think>` text | Reasoning model such as Qwen3 | Already handled by `strip_reasoning`. |

## Status

- Tested end to end against a local mock server that imitates an OpenAI-compatible endpoint, including the embedding fallback and the wrong-token error path. Versions used: Python 3.11, `langgraph` 1.2.11, `langchain-openai` 1.6.2.
- **Not yet run against a real MLIS deployment.** Do one dry run on the lab tenant before class and record the real endpoint format.
- The notebook installs unpinned package versions. Pin them in your notebook image before a class.

## Keep secrets and outputs out of Git

- The notebook never stores the token, but printed outputs include hostnames, working folders and endpoint URLs. Commit the notebook **without outputs**: in JupyterLab use Kernel, Restart Kernel and Clear Outputs of All Cells, then save.
- Never paste a token into a cell or commit a `.env` file.
- Suggested `.gitignore`:

```gitignore
.ipynb_checkpoints/
.venv/
.env
pcai_first_run_*.json
*_run.ipynb
```

## Publish this folder to Git

Run these from this folder. Replace the address with your repository.

```bash
git init
git add README.md pcai_first_notebook.ipynb
git commit -m "Add PCAI first notebook and README"
git branch -M main
git remote add origin <repository URL>
git push -u origin main
```
