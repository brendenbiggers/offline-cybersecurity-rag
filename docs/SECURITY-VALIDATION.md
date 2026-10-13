# Security Validation Report

## Offline Cybersecurity RAG Assistant

**Test environment:** Arch Linux, Lenovo ThinkPad T14 Gen 1  
**Test date:** October 13, 2026  
**Project:** Offline Cybersecurity RAG Assistant  
**Status:** Core functionality verified; network isolation supported by configuration inspection and representative connection testing

---

## 1. Executive Summary

This report documents the security controls implemented and tested during the development of a locally hosted cybersecurity AI assistant.

The system uses Ollama, PrivateGPT, Qwen 2.5 Coder, Nomic Embed Text, and Qdrant to process cybersecurity study documents locally.

The primary security objectives were to:

- Restrict AI services to localhost.
- Prevent AI services from making outbound internet connections.
- Disable Ollama cloud functionality.
- Run application processes under non-root accounts.
- Preserve local communication between the AI components.
- Verify that retrieval-augmented generation remained functional after applying security restrictions.

The testing provided strong evidence that the selected system-level network restrictions functioned as intended in the development environment.

However, the testing did not constitute a comprehensive network-security audit.

## 2. Test Environment

| Component | Configuration |
|---|---|
| Operating system | Arch Linux |
| Hardware | Lenovo ThinkPad T14 Gen 1 |
| Processor | AMD Ryzen 5 PRO |
| Memory | 24 GB |
| Language model | Qwen 2.5 Coder 3B |
| Embedding model | Nomic Embed Text |
| Inference server | Ollama |
| RAG backend | PrivateGPT 1.0.1 |
| Vector database | Local Qdrant |
| Service manager | systemd |
| Host firewall | UFW |

## 3. Test 1 — Localhost API Binding

**Objective:** Verify that the AI services listen on loopback interfaces rather than external network interfaces.

**Command:**

```bash
sudo ss -lntup | grep -E '(:11434|:8080)'
```

**Observed configuration:**

| Service | Listening address |
|---|---|
| Ollama | `127.0.0.1:11434` |
| PrivateGPT | `127.0.0.1:8080` |

**Result:** PASS

Both AI APIs were observed listening on loopback addresses.

**Security significance:** This limits direct exposure of the APIs through external network interfaces.

**Limitation:** Loopback binding does not prevent outbound connections or access by other local processes.

## 4. Test 2 — PrivateGPT Service Isolation

**Objective:** Verify that PrivateGPT runs as a system-level systemd service under an unprivileged Linux account.

**Command:**

```bash
systemctl status cyber-privategpt --no-pager -l
```

**Observed results:**

```text
Active: active (running)
Main PID: 40318 (private-gpt)
CGroup: /system.slice/cyber-privategpt.service
```

The application completed startup and reported:

```text
Uvicorn running on http://127.0.0.1:8080
```

**Result:** PASS

PrivateGPT was running through the intended system-level service rather than an unrestricted terminal-launched background process.

## 5. Test 3 — Active Network Restriction Configuration

**Objective:** Inspect the active systemd IP address restrictions.

**Command:**

```bash
systemctl show cyber-privategpt \
  -p IPAddressDeny \
  -p IPAddressAllow \
  -p User \
  -p ControlGroup \
  --no-pager
```

**Observed results:**

```text
ControlGroup=/system.slice/cyber-privategpt.service
IPAddressAllow=::1/128 127.0.0.0/8
IPAddressDeny=0.0.0.0/0 ::/0
User=brenden
```

**Result:** PASS

The active service configuration showed localhost allowed and other IPv4/IPv6 addresses denied.

**Limitation:** Configuration inspection alone does not prove that packet filtering is enforced. An independent connection test was performed.

## 6. Test 4 — Outbound Network Isolation

**Objective:** Determine whether a system-level systemd process running as a regular Linux user could establish an external TCP connection when subject to the same network restrictions.

**Test command:**

```bash
sudo systemd-run --wait --pipe --collect \
  -p User=brenden \
  -p IPAddressDeny=any \
  -p IPAddressAllow=localhost \
  /usr/bin/python -c 'import socket; s=socket.socket(); s.settimeout(3); print("External connection:", s.connect_ex(("1.1.1.1", 443))); s.close()'
```

**Observed output:**

```text
External connection: 11
```

A nonzero result indicates the attempted connection did not succeed.

The unrestricted comparison test returned:

```text
External connection: 0
```

A return value of `0` indicates a successful connection.

**Result:** PASS for the representative system-level test process.

**Interpretation:** The comparison supports that system-level systemd IP restrictions were effective in the tested environment.

**Limitations:**

- The test used a temporary process with equivalent restrictions rather than initiating the connection from the running PrivateGPT process itself.
- Only one external destination and TCP port were tested.
- This does not prove that every possible network communication path is blocked.
- The test result should not be interpreted as a complete exfiltration-resistance assessment.

## 7. Test 5 — User-Level vs. System-Level Isolation

**Objective:** Compare the behavior of network restrictions applied through user-level and system-level systemd services.

**User-level test result:**

```text
External connection: 0
```

The connection succeeded despite the requested restrictions.

The system journal reported that the unit configured an IP firewall but was not running as root.

**System-level test result:**

```text
External connection: 11
```

The connection failed when equivalent restrictions were applied through a system-level unit running under the regular user account.

**Result:** PASS for identifying and correcting the ineffective configuration approach.

**Lesson learned:** A security setting appearing in a service definition does not guarantee enforcement. Privileges and service-manager context matter.

## 8. Test 6 — Local RAG Functionality

**Objective:** Confirm that the assistant can retrieve indexed cybersecurity documents and generate answers while PrivateGPT runs inside its restricted service.

**Test question:**

> What are the six incident response phases listed in my Security+ notes?

**Observed AI response:**

1. Preparation
2. Detection and Analysis
3. Containment
4. Eradication
5. Recovery
6. Post-Incident Activities

**Retrieved source:**

```text
security-plus-notes.txt
```

**Result:** PASS

The system successfully retrieved relevant content from the local document collection and generated an answer using Qwen.

**Security significance:** The restricted deployment preserved the application's required localhost communication.

**Limitation:** This test confirms functional behavior, not the factual accuracy of all future AI responses.

## 9. Additional Security Controls

### Ollama Cloud Functionality

The Ollama systemd override includes:

```ini
[Service]
Environment="OLLAMA_NO_CLOUD=1"
```

The environment setting was verified in the active service configuration.

### Ollama Network Isolation

The Ollama service uses:

```ini
[Service]
IPAddressDeny=any
IPAddressAllow=localhost
```

The active restrictions were inspected, and a representative system-level connection test was performed.

### Host Firewall

UFW was configured with:

- Incoming traffic denied by default.
- Outgoing traffic allowed by default.

The systemd restrictions provide separate controls specifically for the AI services.

## 10. Findings and Remediation

| Finding | Remediation |
|---|---|
| User-level network restrictions were ineffective | Moved PrivateGPT to a system-level service |
| Original launcher could start PrivateGPT without isolation | Updated launcher to use the restricted systemd service |
| Ollama attempted cloud-related network requests | Disabled cloud functionality and configured IP restrictions |
| Local AI APIs required protection from network exposure | Bound services to loopback interfaces |

## 11. Residual Risks and Future Testing

The following work remains:

- Directly test outbound network access from the PrivateGPT service context.
- Add automated checks for systemd security properties.
- Test multiple network protocols and destinations.
- Assess access permissions on
