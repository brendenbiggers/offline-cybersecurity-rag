# Offline Cybersecurity RAG Assistant

**A privacy-focused, locally hosted AI assistant for cybersecurity research, studying, and document analysis.**

Built on Arch Linux using Ollama, Qwen 2.5 Coder, PrivateGPT, and Qdrant, this project demonstrates how to deploy a retrieval-augmented generation (RAG) system with local inference, document retrieval, Linux service management, and network isolation.

## Project Overview

The goal was to create an AI assistant capable of answering cybersecurity questions using locally stored reference material without relying on a cloud-based language model.

Rather than sending prompts and documents to an external AI service, the system processes queries through locally hosted components.

The project also incorporates defensive security controls designed to restrict outbound network access from its AI services.


## Project Navigation

| Resource | Description |
|---|---|
| [Python RAG Assistant](cyber-rag.py) | Custom Python application for document retrieval and local AI responses |
| [Secure Bash Launcher](scripts/cyber-ai) | Starts the assistant using restricted systemd services |
| [Installation Guide](docs/INSTALLATION.md) | System requirements, configuration, and deployment instructions |
| [Security Validation Report](docs/SECURITY-VALIDATION.md) | Network isolation tests, findings, and verification results |
| [PrivateGPT systemd Service](systemd/cyber-privategpt.service) | Service management and outbound network restrictions |
| [Ollama Security Configuration](systemd/ollama.service.d/) | Network isolation and cloud-disable settings |



## Technology Stack

| Technology | Purpose |
|---|---|
| Arch Linux | Host operating system |
| Ollama | Local model inference and embeddings |
| Qwen 2.5 Coder 3B | Language model for generating answers |
| Nomic Embed Text | Document and query embeddings |
| PrivateGPT | Local RAG backend and API |
| Qdrant | Local vector database |
| Python | Custom command-line assistant |
| Bash | One-command application launcher |
| systemd | Service management and network restrictions |
| UFW | Host firewall configuration |

## System Architecture

```text
                  USER
                   |
                   v
             cyber-ai (Bash)
                   |
                   v
          cyber-rag.py (Python)
                   |
                   v
       PrivateGPT REST API
          127.0.0.1:8080
              /       \
             /         \
            v           v
      Local Qdrant    Ollama API
      Vector Store    127.0.0.1:11434
                         |
                    +----+----+
                    |         |
                    v         v
              Nomic Embed   Qwen 2.5
                  Text      Coder 3B
```

All AI processing and document retrieval are performed locally.

## Key Features

- Local AI inference without requiring cloud-based LLM APIs
- Retrieval-augmented generation using personal cybersecurity study documents
- Persistent local vector storage with Qdrant
- Source references included in generated answers
- Custom Python command-line interface
- Bash launcher for simplified operation
- systemd-managed AI services
- Loopback-only API listeners
- System-level outbound network restrictions
- Cloud functionality disabled in Ollama

## Security Architecture

Security and privacy were important design goals throughout the project.

### Localhost Binding

The services are configured to listen only on loopback interfaces:

- Ollama: `127.0.0.1:11434`
- PrivateGPT: `127.0.0.1:8080`

This prevents direct access to their APIs through the host's external network interfaces.

### systemd Network Isolation

Both AI services use system-level systemd network restrictions:

```ini
[Service]
IPAddressDeny=any
IPAddressAllow=localhost
```

These settings are designed to deny external IPv4 and IPv6 connections while allowing communication between local services.

### Ollama Cloud Restrictions

Ollama is configured with:

```ini
[Service]
Environment="OLLAMA_NO_CLOUD=1"
```

This disables Ollama cloud functionality.

### Service Separation

Ollama runs under its dedicated service account, while PrivateGPT runs under the regular Linux user through a system-level systemd service.

The custom launcher uses the restricted PrivateGPT service instead of directly launching an unrestricted application process.

### Firewall

UFW is configured with:

- Default incoming policy: Deny
- Default outgoing policy: Allow

Application-level network isolation is enforced separately through systemd, allowing normal browser and system internet access.

## RAG Workflow

1. Cybersecurity study documents are added to the local document collection.
2. PrivateGPT processes and indexes the documents.
3. Nomic Embed Text generates embeddings.
4. Qdrant stores the resulting vector representations.
5. The user submits a question through the Python application.
6. PrivateGPT searches for relevant document excerpts.
7. Qwen generates an answer using the retrieved context.
8. The application displays the answer and retrieved source filenames.

## Validation and Testing

### Local Service Verification

Confirmed that Ollama and PrivateGPT were listening on loopback addresses rather than external interfaces.

### Network Isolation Testing

A temporary system-level systemd process was launched with the same network restriction settings used by the PrivateGPT service.

An attempted TCP connection to `1.1.1.1:443` returned a nonzero connection error, indicating the connection failed.

An unrestricted comparison test successfully connected.

This provided evidence that system-level restrictions were effective in the tested environment.

### RAG Functionality Test

A locally indexed Security+ study document was used to test retrieval and answer generation.

**Test question:**

> What are the six incident response phases listed in my Security+ notes?

**Generated answer:**

1. Preparation
2. Detection and Analysis
3. Containment
4. Eradication
5. Recovery
6. Post-Incident Activities

**Retrieved source:**

`security-plus-notes.txt`

The test demonstrated successful document retrieval and local answer generation while PrivateGPT was running through its restricted systemd service.

## Challenges and Lessons Learned

### User-Level vs. System-Level Network Restrictions

During testing, user-level systemd network restrictions did not block the external connection.

The system journal indicated that the user-level service could not enforce the configured IP firewall without the necessary privileges.

Moving PrivateGPT to a system-level service running under the regular user account allowed the same restriction configuration to be applied successfully in a representative test.

### Service Management

Originally, the Bash launcher started PrivateGPT directly as a background process.

This created a potential security gap because the application could run without the intended systemd restrictions.

The launcher was redesigned to start the restricted systemd service instead.

### Local Model Integration

The project required configuring Ollama endpoints, embedding models, PrivateGPT settings, and document retrieval so the components could communicate through local APIs.

## Hardware Environment

- Lenovo ThinkPad T14 Gen 1
- AMD Ryzen 5 PRO processor
- Approximately 24 GB RAM
- Arch Linux

The project demonstrates that a useful local cybersecurity study assistant can operate on relatively affordable, refurbished hardware.

## Future Improvements

- Add automated tests for API availability and document retrieval
- Implement more comprehensive service isolation
- Add structured logging and error handling
- Expand the cybersecurity document collection
- Improve answer quality and source attribution
- Benchmark local model performance
- Create reproducible deployment instructions
- Add screenshots and architecture diagrams

## Responsible Use

This project is intended for cybersecurity education, defensive research, and local document analysis.

Only documents the operator has permission to process should be indexed.

No private study materials, credentials, model weights, or local vector database files are intended to be published in this repository.

## Project Status

**Working prototype — local inference, document retrieval, systemd service integration, and network isolation testing completed.**

Further hardening, automated testing, and deployment documentation are planned.
