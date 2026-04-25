# Ollama + Podman Local Chat Setup Guide

A complete step-by-step tutorial for running Ollama with Podman on Linux Mint, with common errors and solutions.

**Difficulty Level:** Intermediate | **Time:** ~30 minutes | **Prerequisites:** Linux Mint/Ubuntu

---

## 📋 Quick Navigation

- [Prerequisites](#prerequisites)
- [Part 1: Install Ollama](#part-1-install-ollama)
- [Part 2: Create Podman Sandbox](#part-2-create-podman-sandbox)
- [Part 3: Download a Model](#part-3-download-a-model)
- [Part 4: Create and Run Chat Script](#part-4-create-and-run-chat-script)
- [⚠️ Common Errors](./docs/TROUBLESHOOTING.md)
- [Daily Usage](./docs/DAILY_USAGE.md)
- [Cleanup](./docs/CLEANUP.md)

---

## 📋 Prerequisites

Before starting, verify you have the following:

```bash
podman --version
curl --version
df -h /home  # Check free space
```

## Part 1: Install Ollama

### Step 1.1: Download and Install Ollama

```bash
curl -fsSL https://ollama.ai/install.sh | sh
```

**Expected Output:**
```
>>> Installing ollama to /usr/local/bin...
>>> Installing service to /etc/systemd/system/ollama.service...
>>> Downloading latest ollama...
>>> Installation complete. To get started, run ollama pull <model_name>
```

### Step 1.2: Verify Installation

```bash
ollama --version
```

**Expected Output:**
```
ollama version 0.X.X
```

### Step 1.3: Stop the Automatic Ollama Service

⚠️ **Important:** The installer creates a systemd service that runs by default. We'll run Ollama manually instead.

```bash
sudo systemctl stop ollama
sudo systemctl disable ollama
sudo systemctl status ollama
```

**Expected Output:**
```
● ollama.service - Ollama
     Loaded: loaded (...; disabled; ...)
     Active: inactive (dead)
```

## Part 2: Create Podman Sandbox

### Step 2.1: Create Workspace Directories

```bash
mkdir -p ~/ollama-sandbox/models ~/ollama-sandbox/data
ls -la ~/ollama-sandbox
```

**Expected Output:**
```
total 8
drwxr-xr-x  4 user user 4096 Apr 25 12:05 .
drwxr-xr-x 20 user user 4096 Apr 25 12:00 ..
drwxr-xr-x  2 user user 4096 Apr 25 12:05 data
drwxr-xr-x  2 user user 4096 Apr 25 12:05 models
```

### Step 2.2: Download the Containerfile

Copy `scripts/Containerfile` from this repo to `~/ollama-sandbox/Containerfile`

Or create manually:

```bash
cat > ~/ollama-sandbox/Containerfile << 'EOF'
FROM python:3.11-slim

RUN apt-get update && apt-get install -y \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir \
    requests \
    python-dotenv \
    flask

WORKDIR /workspace

VOLUME ["/workspace/models", "/workspace/data"]

CMD ["/bin/bash"]
EOF
```

**Expected Output:** (no output; file created successfully)

### Step 2.3: Build the Podman Image

```bash
cd ~/ollama-sandbox
podman build -t ollama-local-image .
```

**Expected Output:**
```
STEP 1/6: FROM python:3.11-slim
STEP 2/6: RUN apt-get update && apt-get install -y...
...
STEP 6/6: CMD ["/bin/bash"]
COMMIT ollama-local-image
→ 6f1a2b3c4d5e...
```

### Step 2.4: Verify the Image

```bash
podman images | grep ollama-local-image
```

**Expected Output:**
```
localhost/ollama-local-image  latest  6f1a2b3c4d5e  2 min ago  280 MB
```

### Step 2.5: Create and Make the Startup Script Executable

Copy `scripts/run-ollama.sh` from this repo to `~/ollama-sandbox/run-ollama.sh` and run:

```bash
chmod +x ~/ollama-sandbox/run-ollama.sh
```

Or create manually:

```bash
cat > ~/ollama-sandbox/run-ollama.sh << 'EOF'
#!/bin/bash
podman run -it --rm \
  -v ~/ollama-sandbox/models:/workspace/models \
  -v ~/ollama-sandbox/data:/workspace/data \
  --name ollama-local \
  ollama-local-image
EOF

chmod +x ~/ollama-sandbox/run-ollama.sh
```

**Expected Output:** (no output; file created)

## Part 3: Download a Model

### Step 3.1: Find Your Machine's Network IP

```bash
hostname -I
```

**Expected Output:**
```
192.168.1.XXX 172.17.0.1
```

✏️ **Note:** Write down the first IP (in this example: `192.168.1.XXX`) — you'll need it in Step 4.2

### Step 3.2: Start Ollama on Your Host

⚠️ **Critical:** You must bind Ollama to `0.0.0.0` so the container can reach it.

```bash
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

**Expected Output:**
```
time=2026-04-25T12:30:00.123Z level=INFO msg="Listening on 0.0.0.0:11434"
```

✅ **Keep this terminal open.** Open a new terminal for the next steps.

### Step 3.3: Download a Model

On a new terminal on your host (NOT in the container):

```bash
ollama pull mistral
```

**Expected Output:**
```
pulling manifest
pulling 8ee4dff619f3
pulling 541684228241
...
verifying sha256 digest
writing manifest
removing any unused layers
success
```

⏱️ **This takes 5-10 minutes depending on your internet speed (~4 GB download).**

### Step 3.4: Verify the Model Downloaded

```bash
ollama list
```

**Expected Output:**
```
NAME      ID              SIZE      MODIFIED
mistral   6577803aa9a0    4.1 GB    2 minutes ago
```

### Step 3.5: Test Ollama is Reachable on Your Network

```bash
curl http://192.168.1.XXX:11434/api/tags
```

Replace `192.168.1.XXX` with your actual IP from Step 3.1.

**Expected Output:**
```json
{"models":[{"name":"mistral:latest","model":"mistral:latest",...}]}
```

⚠️ **Error?** See [Troubleshooting](#troubleshooting)

## Part 4: Create and Run Chat Script

### Step 4.1: Enter the Podman Container

```bash
~/ollama-sandbox/run-ollama.sh
```

**Expected Output:**
```
root@abc123def456:/workspace#
```

✅ **Your prompt should show you're in the container.**

### Step 4.2: Copy the Chat Script

**IMPORTANT:** Replace `192.168.1.XXX` with your actual IP from Part 3, Step 3.1 before running!

Copy `scripts/ollama_client.py` from this repo to `/workspace/ollama_client.py`

Or create manually inside the container:

```bash
cat > /workspace/ollama_client.py << 'EOF'
#!/usr/bin/env python3
import os
import json
import requests
from datetime import datetime

OLLAMA_API = "http://192.168.1.XXX:11434/api/generate"
CONVERSATION_FILE = "/workspace/data/conversation.json"
MODEL = "mistral"

def save_conversation(history):
    with open(CONVERSATION_FILE, 'w') as f:
        json.dump(history, f, indent=2)

def load_conversation():
    if os.path.exists(CONVERSATION_FILE):
        with open(CONVERSATION_FILE, 'r') as f:
            return json.load(f)
    return []

def chat_with_ollama():
    conversation_history = load_conversation()
    
    print("Ollama Local Chat (Podman + Persistent Storage)")
    print(f"Model: {MODEL}")
    print(f"Conversation saved to: {CONVERSATION_FILE}")
    print("Type 'exit' to quit, 'clear' to start fresh\n")
    
    while True:
        user_input = input("You: ").strip()
        
        if user_input.lower() == 'exit':
            print("Saving and exiting...")
            save_conversation(conversation_history)
            break
        
        if user_input.lower() == 'clear':
            conversation_history = []
            print("Conversation cleared.\n")
            continue
        
        if not user_input:
            continue
        
        conversation_history.append({
            "role": "user",
            "content": user_input,
            "timestamp": datetime.now().isoformat()
        })
        
        context = "\n".join([
            f"{msg['role'].capitalize()}: {msg['content']}"
            for msg in conversation_history
        ])
        
        try:
            print("\nOllama: ", end="", flush=True)
            response = requests.post(
                OLLAMA_API,
                json={
                    "model": MODEL,
                    "prompt": context,
                    "stream": False
                },
                timeout=300
            )
            
            if response.status_code == 200:
                result = response.json()
                assistant_message = result.get('response', '').strip()
                print(assistant_message)
                
                conversation_history.append({
                    "role": "assistant",
                    "content": assistant_message,
                    "timestamp": datetime.now().isoformat()
                })
                
                save_conversation(conversation_history)
                print()
            else:
                print(f"Error: {response.status_code}")
        
        except requests.exceptions.ConnectionError:
            print("ERROR: Cannot connect to Ollama API on 192.168.1.XXX:11434")
            print("Make sure Ollama is running on your host with:")
            print("OLLAMA_HOST=0.0.0.0:11434 ollama serve")
            break
        except Exception as e:
            print(f"ERROR: {e}")

if __name__ == "__main__":
    chat_with_ollama()
EOF

chmod +x /workspace/ollama_client.py
```

### Step 4.3: Verify the Script Was Created

```bash
ls -la /workspace/ollama_client.py
```

**Expected Output:**
```
-rwxr-xr-x 1 root root 2345 Apr 25 12:35 /workspace/ollama_client.py
```

### Step 4.4: Test the Connection to Ollama

```bash
python -c "import requests; print(requests.get('http://192.168.1.111:11434/api/tags').json())"
```

Replace `192.168.1.111` with your actual IP.

**Expected Output:**
```python
{'models': [{'name': 'mistral:latest', ...}]}
```

⚠️ **Error?** See [Troubleshooting](#troubleshooting)

### Step 4.5: Run the Chat

```bash
python /workspace/ollama_client.py
```

**Expected Output:**
```
Ollama Local Chat (Podman + Persistent Storage)
Model: mistral
Conversation saved to: /workspace/data/conversation.json
Type 'exit' to quit, 'clear' to start fresh

You: 
```

### Step 4.6: Test a Question

```
You: What is Podman?
```

**Expected Output (takes 10-30 seconds on first run):**
```
Ollama: Podman is a daemonless container engine for running containers without requiring a running daemon service...

You:
```

Type `exit` to quit.

### Step 4.7: Verify Data Persistence

```bash
exit
```

Back on your host machine, check the saved conversation:

```bash
cat ~/ollama-sandbox/data/conversation.json
```

**Expected Output:**
```json
[
  {
    "role": "user",
    "content": "What is Podman?",
    "timestamp": "2026-04-25T12:30:45.123456"
  },
  {
    "role": "assistant",
    "content": "Podman is a daemonless container engine...",
    "timestamp": "2026-04-25T12:30:60.654321"
  }
]
```

✅ **Success! Your data persists on disk.**

## Troubleshooting

### Error: `listen tcp 0.0.0.0:11434: bind: address already in use`

**Cause:** Ollama is already running (likely from the systemd service).

**Solution:**
```bash
sudo systemctl stop ollama
sudo systemctl disable ollama
pkill -9 ollama
sleep 2
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

### Error: `curl: (7) Failed to connect to 192.168.1.111 port 11434`

**Cause:** Ollama isn't running or isn't listening on all interfaces.

**Solution:**

1. Check if Ollama is running on your host:
   ```bash
   ps aux | grep "ollama serve"
   ```

2. If not running, start it:
   ```bash
   OLLAMA_HOST=0.0.0.0:11434 ollama serve
   ```

3. If you started it without `OLLAMA_HOST=0.0.0.0:11434`, it only listens on localhost. Stop and restart it correctly:
   ```bash
   pkill ollama
   sleep 2
   OLLAMA_HOST=0.0.0.0:11434 ollama serve
   ```

4. Test locally first:
   ```bash
   curl http://127.0.0.1:11434/api/tags
   ```

### Error: `curl: ... returned empty models list or {"models":null}`

**Cause:** Ollama is running but no models are downloaded.

**Solution:**
```bash
ollama pull mistral
```

Wait for it to finish. Monitor progress with:
```bash
ollama list
```

### Error: Cannot connect to Ollama API (from inside container)

**Cause:** One of three issues:

1. Ollama isn't running on your host
2. You used the wrong IP address
3. Firewall is blocking port 11434

**Solution on your host:**

1. Verify Ollama is running:
   ```bash
   curl http://127.0.0.1:11434/api/tags
   ```

2. Check your IP:
   ```bash
   hostname -I
   ```

3. Test from your host on the network IP:
   ```bash
   curl http://192.168.1.XXX:11434/api/tags
   ```
   (Replace XXX with your actual IP)

## 📚 Additional Resources

- [Ollama Documentation](https://ollama.ai)
- [Podman Documentation](https://podman.io)
- [Common Errors & Solutions](#troubleshooting)

## 📝 License

This tutorial is licensed under the MIT License — see the LICENSE file for details.

## 🤝 Contributing

Found an issue or have improvements? Feel free to open a pull request or issue!

## ⭐ If This Helped You

Please give this repo a star! It helps others discover this tutorial.
```

This README.md includes:
- ✅ All tutorial content formatted properly with Markdown headings
- ✅ Code blocks with bash and JSON syntax highlighting
- ✅ Clear section organization with Parts 1-4
- ✅ Expected output for each step
- ✅ Integrated troubleshooting section
- ✅ Emoji headers for quick scanning
- ✅ Links between sections (e.g., "See Troubleshooting")
- ✅ License, contributing, and star sections at the bottom
