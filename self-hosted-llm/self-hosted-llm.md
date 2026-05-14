# Guide on setting up self-hosted LLM
## Ollama
Install ollama:
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Download and run a model:
```bash
ollama run qwen2.5-coder:32b
```

Enable llama as a background service. This can be done two ways:
  a) Using `systemctl` (sometimes edits do not persist):
```bash
sudo systemctl edit ollama.service
```
and simply add:
```TOML
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
```

  b) An override:
```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
echo -e "[Service]\nEnvironment=\"OLLAMA_HOST=0.0.0.0\"" | sudo tee /etc/systemd/system/ollama.service.d/override.conf
```

With each method, you need to reload & restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## WebUI
Assumption: Docker is installed

Simply download and run the docker container:
```bash
sudo docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

If the ip used above is already taken by something else, try again:
```bash
sudo docker rm open-webui  # Remove old
sudo docker run -d -p 3001:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:mainsudo docker run -d -p 3001:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

## Profit!
Simply access your model in your favorite browser on "http://localhost:3001". Note the IP at the end (i.e. 3001 in this case) should match the one you set in the [WebUI](WebUI) section.

For access from a different machine, you might wanna look into either ssh tunnling, smth like:
```bash
ssh -L 3001:localhost:3001 your_username@remote_ubuntu_ip
```
or setting up something slightly safer like WireGuard - see setup for WireGuard (TODO: Add the setup for wg into this repo)

# Updates
## Ollama
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

## WebUI
This runs in a docker container so we need to create a new one.
Luckily, smart people have designed this so that the chat history lives elsewhere.
```bash
sudo docker rm -f open-webui
sudo docker pull ghcr.io/open-webui/open-webui:main
sudo docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```
