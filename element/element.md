# Element + Matrix server
Assumptions: Docker installed, ubuntu/debian-based system, internet domain (say "yourdomain.com" for the sake of this guide) ownership (see, e.g., [cloudflare](https://www.cloudflare.com/products/registrar/)).

## Basic setup
Create the required directories and go inside:
```bash
mkdir -p ~/matrix-server/synapse-data
mkdir -p ~/matrix-server/caddy-data
cd ~/matrix-server
```

Run synapse server:
```bash
docker run -it --rm \
    -v "$PWD/synapse-data:/data" \
    -e SYNAPSE_SERVER_NAME=yourdomain.com \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest generate
```

You will need to match your domain inside the config yaml file: `nano synapse-data/homeserver.yaml` to be:
```YAML
server_name: yourdomain.com
```
Also, disable registrations (if so you wish, but likely yes). We will still be able to add users through the terminal - this will simply disable the gui registrations.

Next, we create the docker compose file:
```bash
nano docker-compose.yml
```
populate it with:
```YAML
services:
  # The Matrix Server
  synapse:
    image: matrixdotorg/synapse:latest
    container_name: synapse
    restart: unless-stopped
    volumes:
      - ./synapse-data:/data
    environment:
      - SYNAPSE_SERVER_NAME=yourdomain.com # REPLACE THIS
      - SYNAPSE_REPORT_STATS=yes

  # The Element web client
  element:
    image: vectorim/element-web:latest
    container_name: element
    restart: unless-stopped
    volumes:
      - ./element-config.json:/app/config.json

  # The web server (reverse proxy) & HTTPS manager
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./caddy-data:/data
      - ./caddy-config:/config
    depends_on:
      - synapse
      - element
```

Now, we need to configure element to talk to our server. Add the following into the `element-config.json` file:
```JSON
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://matrix.yourdomain.com",
            "server_name": "yourdomain.com"
        },
        "m.identity_server": {
            "base_url": "https://vector.im"
        }
    },
    "brand": "My Element"
}
```

Configure Caddy (`CaddyFile` file):
```
# The matrix server
matrix.yourdomain.com {
    reverse_proxy synapse:8008
}

# The Element web client
chat.yourdomain.com {
    reverse_proxy element:80
}

# Federation (optional: This allows to make it so that other servers can find you)
yourdomain.com {
    header /.well-known/matrix/server "{\"m.server\": \"matrix.yourdomain.com:443\"}"
    header /.well-known/matrix/client "{\"m.homeserver\": {\"base_url\": \"https://matrix.yourdomain.com\"}}"
}
```

Setup DNS - add the following records:
- "matrix"
- "chat"
and make them both point to your home IP (i.e. the public IPv4 ip of the machine you want to host this setup on). This should create two "new" domains - "https://chat.yourdomain.com" is the one you care about (the one with the GUI), "https://matrix.yourdomain.com" is the backend.

Almost there. If we want users outside our local network to be able to chat with us using this setup, we need to setup port forwarding. Do this at your own risk! (Installing WireGuard first helps protect your traffic. see: [WireGuard setup guide](../wireguard/wireguard.md)).
We need two rules:
- TCP port 80 to host local IP
- TCP port 443 to host local IP
You can find your local ip by running your favourite terminal command, e.g. `ip addr` or `hostname -I`.

Finally, launch everything:
```bash
docker compose up -d
```

## Creating accounts
Run in the terminal:
```bash
docker exec -it synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
```
And simply follow the process (at the end it asks for whether the user should be admin or not).


## Android setup
Download the [Elemenet Classic](https://play.google.com/store/apps/details?id=im.vector.app) app (or the other one, but I like this one better).

By default the app will want to use the `matrix.org` server, we need to change this to point to our matrix instance: `https://matrix.yourdomain.com`. 

Now you can simply login using the your username and password.

## Connecting to accounts on other matrix servers
Use the UI. If the username on your local matrix instance is "martin", your full username will be "@martin:matrix.yourdomain.com" and you should use this for to connect with other people (otherwise, they will not be able to find you easily or at all.)
