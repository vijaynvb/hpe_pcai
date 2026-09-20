# HPE Private Cloud AI: first notebook

Code for the training "Agentic AI: Building and Shipping Production Agent Systems", delivered on HPE Private Cloud AI (PCAI) through HPE GreenLake.

This folder holds a notebook that proves a notebook server on PCAI can reach a model served by HPE MLIS (Model Endpoints in HPE AI Essentials 1.9.x), then runs a small LangGraph agent on it. It is the code for Lab 0 in Module 1.

## Contents

| File | What it is |
|---|---|
| `pcai_first_notebook.ipynb` | The notebook. Nine steps, run one cell at a time. |
| `README.md` | This file. |

## What the notebook does

1. Prints where the kernel runs (Python version, hostname, working folder).
2. Installs `langchain-openai`, `langgraph` and `requests` with `%pip`.
3. Reads the endpoint URLs and API tokens from environment variables, or asks for them (tokens with a hidden prompt). The URLs are pre-filled with the lab endpoints `llm-llama8b-1` and `embedder-llama8b-1`; press Enter to accept them. Step 3b then makes Python trust the platform's certificate: your CA file, else the notebook server's system certificates, else, for lab tenants only, verification off.
4. Asks the endpoint which model it serves (`GET <api root>/models`, trying `<endpoint>/v1` and then `<endpoint>`), then makes one raw `POST` to `<api root>/chat/completions`. The token is sent as an `Authorization: Bearer` header (omitted if no token was entered).
5. Makes the same call through LangChain (`ChatOpenAI` with a custom `base_url`). MLIS endpoints are OpenAI-compatible, so this class is only the client. The request goes to MLIS, not to OpenAI.
6. Optionally calls the embedding endpoint at `<api root>/embeddings`, detecting the model the same way.
7. Runs a two-path LangGraph graph for IT incidents: `classify`, then `escalate` for high severity or `resolve` otherwise. The routing decision is plain code on a value in the state.
8. Saves the results to `pcai_first_run_<date>_<time>.json` in the working folder. The token is never written.
9. Wrap-up and a short troubleshooting table.

## Run it on PCAI

1. Sign in to HPE AI Essentials and open **Notebooks**.
2. Start your notebook server (or create one with JupyterLab and an image that has Python 3.10 or later), then connect.
3. Upload `pcai_first_notebook.ipynb` through the JupyterLab file browser, or `git clone` this repository from a notebook terminal if the network allows it.
4. Go to **Gen AI, Model Endpoints**. Check that the endpoints you need show **Ready**, and copy the **Endpoint** value (an endpoint that is still **Deploying** has no URL). No model name is needed: the notebook detects it. Open the endpoint: if **Authentication** is **Yes**, click **Generate API Token** under *API Token Management* and copy the token when it is shown. Each endpoint has its own tokens.
5. Open the notebook, choose the **Python 3** kernel, and run each cell with `Shift + Enter`. Enter the values when prompted.
6. When finished, stop the notebook server from the Notebook Servers screen.

Menu names follow HPE AI Essentials 1.9.x, the version deployed for this training. Compare them with your screen.

## Settings

Each setting is read from an environment variable if that variable exists. Otherwise the notebook prompts for it.

| Variable | Required | Meaning |
|---|---|---|
| `MLIS_LLM_BASE_URL` | No | Endpoint of the LLM. Default: the lab `llm-llama8b-1` endpoint. A trailing `/chat/completions` is trimmed, and `/v1` is added automatically when the endpoint needs it. |
| `MLIS_LLM_MODEL` | No | Model name. Empty means detect it from `GET /models`. Set it to force one model when an endpoint serves several. |
| `MLIS_DEPLOY_TOKEN` | Yes, if the endpoint shows Authentication: Yes | API token for the LLM endpoint, sent as `Authorization: Bearer <token>`. Empty means no header is sent. |
| `MLIS_EMB_BASE_URL` | No | Endpoint of the embedding model. Default: the lab `embedder-llama8b-1` endpoint. Set it to an empty value (or type `skip` at the prompt) to skip Step 6. |
| `MLIS_EMB_MODEL` | No | Embedding model name. Empty means detect it. |
| `MLIS_EMB_TOKEN` | If the embedding endpoint needs one | API token for the embedding endpoint. Unset means the notebook asks; empty means reuse the LLM token. |
| `MLIS_CA_BUNDLE` | No | Path to a PEM/CRT file with the platform's certificate authority (Step 3b). Recommended when Python reports `CERTIFICATE_VERIFY_FAILED`. If unset, a file named `pcai-ca.pem` or `pcai-ca.crt` next to the notebook is used automatically. Give it the full chain (intermediate and root CA). |
| `MLIS_VERIFY_SSL` | No | `false` switches certificate verification off. Lab or training tenants only: the token could be sent to an impostor. Default `true`. |

The defaults in the notebook (`DEFAULT_LLM_URL`, `DEFAULT_EMB_URL` in Step 3) point at the endpoints in namespace `project-user-kiran-kumar-m`. Edit them, or set the variables, to use another namespace.

## Run it headless

Useful for a health check before a session. Run in a terminal where `jupyter nbconvert` is installed.

```bash
export MLIS_LLM_BASE_URL="<endpoint URL>"
read -s -p "API token: " MLIS_DEPLOY_TOKEN; export MLIS_DEPLOY_TOKEN; echo
export MLIS_CA_BUNDLE="<path to CA file>"   # or: export MLIS_VERIFY_SSL=false (lab only)
export MLIS_EMB_BASE_URL=""        # empty skips the embedding step; or set the embedding endpoint URL

jupyter nbconvert --to notebook --execute pcai_first_notebook.ipynb \
    --output pcai_first_notebook_run.ipynb
```

The executed copy contains outputs such as endpoint addresses. Do not commit it.

## Run it outside PCAI

Any OpenAI-compatible chat endpoint works, including MLIS endpoints reachable from your machine.

```bash
python -m venv .venv
source .venv/bin/activate
pip install jupyterlab langchain-openai langgraph requests
export MLIS_LLM_BASE_URL="<endpoint URL>" MLIS_DEPLOY_TOKEN="<token, or empty>"
jupyter lab
```

## Troubleshooting

| What you see | Likely cause | What to do |
|---|---|---|
| `HTTP 401` or `403` | No token, a wrong token, or an expired one. The endpoint page shows **Authentication: Yes** | Click **Generate API Token** on the endpoint page and enter it in Step 3. |
| `HTTP 404`, or "Could not reach the endpoint" | Wrong URL, or the endpoint is not Ready | Copy the **Endpoint** value again from Gen AI, Model Endpoints. The notebook already tries `/v1` for you. |
| "listed no models" | The endpoint does not implement `GET /models` | Set `MLIS_LLM_MODEL` (or `MLIS_EMB_MODEL`) to the model name. |
| `HTTP 502`, `503` or a timeout | Deployment starting, scaled to zero, or busy | Check its status in MLIS, wait and retry. |
| `CERTIFICATE_VERIFY_FAILED`, "unable to get local issuer certificate" | The platform's internal certificate authority is not trusted by Python. This happens before any HTTP request, so it is not a token problem. | Step 3b stops with a message until you choose: drop the CA file next to the notebook as `pcai-ca.pem` (or set `CA_BUNDLE`), or in a lab tenant only set `VERIFY_SSL = False`. Then re-run Step 3b. |
| `ModuleNotFoundError` | Kernel restarted, so `%pip` installs in the base environment were removed | Re-run Step 2, restart the kernel, continue. |
| Answer contains `<think>` text | Reasoning model such as Qwen3 | Already handled by `strip_reasoning`. |

## Status

- Tested end to end against a local mock server that imitates a NIM-style MLIS endpoint (API only under `/v1`, root returns 404), covering: no token, a required token, a wrong token, a URL pasted with `/v1/chat/completions`, model auto-detection, and a dead URL. Versions used: Python 3.11, `langgraph` and `langchain-openai` current at the time of testing.
- Also tested against an HTTPS mock signed by a private CA: the notebook reproduces `unable to get local issuer certificate` without Step 3b, and runs end to end with `CA_BUNDLE`, with the CA in the system store, and with `VERIFY_SSL = False`.
- A first run against the real `llm-llama8b-1` endpoint reached the certificate error above. Not yet confirmed after the Step 3b fix: that the endpoint answers `GET /v1/models` and `POST /v1/chat/completions` with the API token. Note that `llm-llama8b-1` reports its model as `meta/llama-3.2-1b-instruct`; a 1B model can misjudge incident severity in Step 7.
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
