# The Ultimate Private AI Stack Guide

This guide allows you to host your own "ChatGPT" (Open WebUI) on your own hardware, expose it securely to the internet via Cloudflare (without opening router ports), and keep it automatically updated.

**The Stack:**
1.  **Open WebUI + Ollama:** The AI interface and inference engine (bundled).
2.  **Watchtower:** Auto-updates your containers.
3.  **Cloudflare Tunnel:** Secure remote access.

---

## Phase 1: Server Preparation

### 1. Prerequisites
* **OS:** Linux (Ubuntu 22.04+ recommended).
* **Domain:** A domain name managed by Cloudflare (e.g., `example.com`).
* **Hardware:** An NVIDIA GPU is highly recommended for performance.

### 2. Install Docker
Run the following to install the latest Docker version:

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker:
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3. Install NVIDIA Drivers (GPU Users Only)
If you have an NVIDIA card, you must install the Container Toolkit so Docker can see it.

```bash
# Add repositories
curl -fsSL [https://nvidia.github.io/libnvidia-container/gpgkey](https://nvidia.github.io/libnvidia-container/gpgkey) | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L [https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list](https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list) | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Install and Configure
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

## Phase 2: Secure Access (Cloudflare Tunnel)

1.  Log in to **[Cloudflare Zero Trust](https://one.dash.cloudflare.com/)**.
2.  Go to **Networks > Tunnels** → **Create a Tunnel**.
3.  Select **Cloudflared** → **Next**.
4.  Name it (e.g., `ai-server`) → **Save**.
5.  **Select Environment:** Docker.
6.  **Copy Token:** Copy the long text string following `--token`. **Save this for Phase 3.**
7.  **Configure Hostname:**
    * **Subdomain:** `ai` (or your preference).
    * **Domain:** `example.com` (Select your domain).
    * **Service Type:** `HTTP`.
    * **URL:** `localhost:3000`.
8.  Click **Save Tunnel**.

---

## Phase 3: The Master Compose File

Create a file named `docker-compose.yml` on your server.
**Important:** Replace `<YOUR_CLOUDFLARE_TOKEN>` with the token from Phase 2.

```yaml
services:
  # 1. The AI Brain & Interface
  open-webui:
    image: ghcr.io/open-webui/open-webui:ollama
    container_name: open-webui
    restart: always
    ports:
      - "3000:8080"
    volumes:
      - ollama_data:/root/.ollama
      - open_webui_data:/app/backend/data
    environment:
      - ENABLE_API_KEYS=true # Forces API keys to be enabled
      # - OLLAMA_BASE_URL=[http://host.docker.internal:11434](http://host.docker.internal:11434) # Uncomment if using external Ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  # 2. The Auto-Updater
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_CLEANUP=true         # Removes old images to save space
      - WATCHTOWER_POLL_INTERVAL=43200  # Checks every 12 hours

  # 3. The Secure Tunnel
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: always
    network_mode: host # Critical: allows it to see localhost:3000
    command: tunnel --no-autoupdate run --token <YOUR_CLOUDFLARE_TOKEN>

volumes:
  ollama_data:
  open_webui_data:
```

### Start the Stack
```bash
sudo docker compose up -d
```

---

## Phase 4: API Configuration (Critical)

To use your AI with apps (Cursor, VS Code, n8n), you must configure permissions.

### 1. Allow API Traffic (Cloudflare)
*Cloudflare Access blocks API bots by default. You must bypass the email check for API paths.*
1.  Go to **Cloudflare Zero Trust > Access > Applications**.
2.  Edit your AI application.
3.  **Policies** tab → **Add a policy**.
4.  **Action:** `Bypass`.
5.  **Rule:** Selector `Path` → Value `/api*`.
6.  Save Policy.

### 2. Generate API Key (Open WebUI)
1.  Go to your URL (e.g., `https://ai.example.com`).
2.  **Sign Up** (First user is Admin).
3.  **Download a Model:** Settings > Models > Pull `llama3.2`.
4.  **Enable User Permissions (The "Hidden" Step):**
    * Go to **Admin Panel > Users > Groups**.
    * Edit **Default Permissions** (Pencil Icon).
    * **Features** tab → Toggle **API Keys** ON → Save.
5.  **Get Key:**
    * Go to **Settings > Account**.
    * Click **Generate New API Key**.

---

## Phase 5: Connecting External Apps

Use these settings in any OpenAI-compatible app:

* **Base URL:** `https://ai.example.com/api` (or `/api/v1`)
* **API Key:** `sk-xxxx...` (Your generated key)
* **Model:** `llama3.2` (Must match the downloaded model name)

---

## Phase 6: User Guide - How to Chat

### 1. Model Selection
* **Top Left Dropdown:** Pick your model.
* **Compare Mode:** Click the **`+`** button to load two models at once and see who answers better.

### 2. Chat with Files (RAG)
* **Upload:** Click **`+`** (left of chat input) > Upload Files.
* **Quick Attach:** Type **`#`** in the chat box to instantly select an uploaded file.
    * *Usage:* Attach a PDF and ask "Summarize the key points."

### 3. Web Browsing
* Type **`#`** followed by a URL.
    * *Example:* `#https://bbc.com/news What are the headlines?`

### 4. Shortcuts
* **Voice Chat:** Click the Headphone icon.
* **History:** `Ctrl + Shift + S` toggles the sidebar.
* **Edit Message:** Press `Up Arrow` to edit your last prompt.
