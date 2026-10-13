# Installation and Deployment Guide

## Offline Cybersecurity RAG Assistant

This guide documents the deployment architecture and configuration used to build a privacy-focused cybersecurity AI assistant on Arch Linux.

**Project status:** Working prototype. The components were validated on the original development machine. This is not yet a fully automated, independently tested installation procedure.

## 1. System Requirements

### Tested Hardware

- Lenovo ThinkPad T14 Gen 1
- AMD Ryzen 5 PRO processor
- 24 GB RAM
- Arch Linux

### Software Components

- Python 3
- Ollama
- PrivateGPT 1.0.1
- Qwen 2.5 Coder 3B
- Nomic Embed Text
- Qdrant (local storage)
- systemd
- curl
- Git

A Linux distribution with systemd is recommended for reproducing the service configuration.

## 2. Project Structure

```text
offline-cybersecurity-rag/
├── README.md
├── .gitignore
├── cyber-rag.py
├── scripts/
│   └── cyber-ai
├── systemd/
│   ├── cyber-privategpt.service
│   └── ollama.service.d/
│       ├── privacy.conf
│       └── no-cloud.conf
└── docs/
    └── INSTALLATION.md
```

Private documents, model weights, and vector database files are intentionally excluded.

## 3. Local AI Models

The project uses Ollama to provide local language-model inference and embeddings.

After installing and starting Ollama, download the required models:

```bash
ollama pull qwen2.5-coder:3b
ollama pull nomic-embed-text
```

These downloads require internet access during initial setup.

Once the models and software dependencies are installed locally, the assistant can perform inference and document retrieval without a cloud AI service.

Verify installed models:

```bash
ollama list
```

## 4. PrivateGPT Configuration

PrivateGPT provides the local RAG API.

The tested development environment used PrivateGPT 1.0.1 installed through `uv tool`.

PrivateGPT must be configured to use:

- Ollama as its local LLM provider
- Qwen 2.5 Coder 3B as the default language model
- Nomic Embed Text as the embedding model
- Qdrant as the local vector database
- Localhost-only API communication
- Offline operation

The development environment used a PrivateGPT override configuration stored outside the Git repository.

That private configuration is not included in this project because installation paths and runtime settings are environment-specific.

**Reproduction limitation:** PrivateGPT installation and its complete override configuration must be established separately. A verified portable configuration template is planned.

## 5. Deploy the PrivateGPT systemd Service

The repository includes:

`systemd/cyber-privategpt.service`

Before installing the service, replace `YOUR_USERNAME` with the intended local Linux account and verify all paths.

Install the configured service file:

```bash
sudo cp systemd/cyber-privategpt.service /etc/systemd/system/
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Start the service:

```bash
sudo systemctl start cyber-privategpt.service
```

Check its status:

```bash
systemctl status cyber-privategpt.service --no-pager
```

The expected result is an active service with PrivateGPT listening on `127.0.0.1:8080`.

The original development machine used manual service startup rather than automatic startup at boot.

## 6. Configure Ollama Network Isolation

The repository includes two systemd drop-in configurations:

- `systemd/ollama.service.d/privacy.conf`
- `systemd/ollama.service.d/no-cloud.conf`

Create the systemd drop-in directory if necessary:

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
```

Copy the configuration files:

```bash
sudo cp systemd/ollama.service.d/*.conf /etc/systemd/system/ollama.service.d/
```

Reload and restart Ollama:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

The privacy configuration contains:

```ini
[Service]
IPAddressDeny=any
IPAddressAllow=localhost
```

The cloud configuration contains:

```ini
[Service]
Environment="OLLAMA_NO_CLOUD=1"
```

These settings restrict network access for the Ollama service and disable its cloud functionality.

**Important:** Download models and complete any required online installation steps before enabling the network restrictions.

## 7. Verify Local API Binding

Check the listening addresses:

```bash
sudo ss -lntup | grep -E '(:11434|:8080)'
```

Expected API endpoints:

| Component | Address |
|---|---|
| Ollama | `127.0.0.1:11434` |
| PrivateGPT | `127.0.0.1:8080` |

Both services should listen only on loopback interfaces.

## 8. Verify systemd Network Restrictions

Inspect the active PrivateGPT configuration:

```bash
systemctl show cyber-privategpt \
  -p IPAddressDeny \
  -p IPAddressAllow \
  -p User \
  -p ControlGroup \
  --no-pager
```

Expected restrictions:

```text
IPAddressAllow=::1/128 127.0.0.0/8
IPAddressDeny=0.0.0.0/0 ::/0
```

These properties confirm the configured address restrictions.

A separate connection test is necessary to evaluate enforcement.

## 9. Prepare the Document Collection

The Python application expects a PrivateGPT collection named:

`pgpt_collection`

Documents must be ingested and indexed into the local PrivateGPT/Qdrant knowledge base before retrieval will work.

The original project used a locally stored Security+ study document for validation.

Private documents and local Qdrant storage are not included in this repository.

A reproducible document-ingestion guide is planned.

## 10. Launch the Assistant

The repository includes:

- `cyber-rag.py` — Python command-line application
- `scripts/cyber-ai` — Bash launcher

Make the launcher executable:

```bash
chmod +x scripts/cyber-ai
```

Run it from the repository:

```bash
./scripts/cyber-ai
```

The launcher checks Ollama availability, starts PrivateGPT through its restricted systemd service when necessary, waits for the API, and starts the Python assistant.

Depending on the local configuration, starting PrivateGPT may require a sudo password.

## 11. Functional Validation

Once the document collection is populated, ask a question based on an indexed document.

Example:

```text
What are the six incident response phases listed in my Security+ notes?
```

The tested system returned:

1. Preparation
2. Detection and Analysis
3. Containment
4. Eradication
5. Recovery
6. Post-Incident Activities

Retrieved source:

`security-plus-notes.txt`

This demonstrated successful retrieval and answer generation using locally hosted services.

## 12. Troubleshooting

### Ollama is unavailable

Check:

```bash
systemctl status ollama --no-pager
```

Verify the local API:

```bash
curl http://127.0.0.1:11434/api/tags
```

### PrivateGPT fails to start

Check:

```bash
systemctl status cyber-privategpt --no-pager
```

Inspect logs:

```bash
journalctl -u cyber-privategpt -n 50 --no-pager
```

Verify the PrivateGPT installation, environment configuration, and filesystem permissions.

### No relevant documents found

Confirm that documents have been indexed into the expected collection and that the embedding model is configured correctly.

### External connections are blocked

This is expected for the restricted AI services. Model downloads and package installation should be completed before enabling network isolation.

## 13. Security Considerations

- Do not commit passwords, API tokens, SSH keys, or private documents.
- Keep AI API listeners bound to localhost.
- Run application services without unnecessary root privileges.
- Validate systemd restrictions rather than assuming they are enforced.
- Treat retrieved documents as untrusted input.
- Avoid indexing confidential material without authorization.
- Remember that local inference alone does not guarantee complete system privacy.

## Future Deployment Improvements

- Publish a verified PrivateGPT configuration template
- Document reproducible document ingestion
- Add automated installation checks
- Add automated network-isolation tests
- Test deployment on a clean Linux installation
- Add screenshots of the working application

---

**Environment:** Arch Linux / ThinkPad T14 Gen 1  
**Project type:** Defensive cybersecurity, Linux administration, local AI, retrieval-augmented generation
