# Setup guide for self-hosted Servarr architecture sufficient to replace Netflix & Spotify
Note: This guide is for educational purposes only. Please support your favourite artists instead of pirating their content.
Assumptions: Ubuntu/Debian-based OS. Docker installed. [wireguard](./../wireguard/wireguard.md) setup studied.

## VPN
For this, we would like our traffic data protected, our real IP address hidden, etc. For this, we use a VPN. In this setup, we use [NordVPN](https://nordvpn.com/?srsltid=AfmBOoqoxWAfoq7SxqnaVSKt5LGy244ap835G7Az_7VKqE7qSLl3G0DW) (because that is the VPN I like) but of course this is flexible.

Log into your VPN account (in a browser) and generate an "Access Token" (usually near "Manual setup" section). Save this in a safe place - we will need it later.

Install the necessary packages:
```bash
sudo apt update && sudo apt install jq curl
```

Run the following command to generate your VPN private key:
```bash
curl -s -u token:YOUR_TOKEN https://api.nordvpn.com/v1/users/services/credentials | jq -r .nordlynx_private_key
```
Above, "YOUR_TOKEN" should be replaced by your "Access Token" from earlier. Store the resulting "NordVPN key"" from this command in also a safe place.

## Directory structure
Feel free to adjust to your liking, but note that if you make changes, they will need to be reflected later on in this guide.
```bash
# Create the bulk storage directories
sudo mkdir -p /data/{torrents,media/{movies,tv,music}}

# Create the application configuration directories
sudo mkdir -p /opt/appdata/{jellyfin,navidrome,jellyseerr,radarr,sonarr,lidarr,prowlarr,gluetun,qbittorrent,flaresolverr}

# Take ownership so docker can manage them
sudo chown -R $USER:$USER /data
sudo chown -R $USER:$USER /opt/appdata
```

## Docker setup
We will run the entire stack using Docker. In this guide, we put the docker `yml` files into the `/home/$USER/` folder (feel free to change this). 
The infrastructure will be split into two parts:
 - Inside the VPN vault (servarr here)
 - Outside the VPN vault (media delivery here, e.g. jellyfin & jellyseerr)

### Inside VPN vault
We name the docker `yml` file for this part `servarr-compose.yml`. Create this file and paste the following contents inside:
```YAML
version: "3.8"
services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    environment:
      - VPN_SERVICE_PROVIDER=nordvpn
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=YOUR_NORDVPN_KEY_HERE
      - SERVER_COUNTRIES=Netherlands
    volumes:
      - /opt/appdata/gluetun:/gluetun
    ports:
      - 8080:8080 # qBittorrent
      - 7878:7878 # Radarr
      - 8989:8989 # Sonarr
      - 8686:8686 # Lidarr
      - 9696:9696 # Prowlarr
      - 8191:8191 # FlareSolverr
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
      - WEBUI_PORT=8080
    volumes:
      - /opt/appdata/qbittorrent:/config
      - /data/torrents:/data/torrents
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - /opt/appdata/radarr:/config
      - /data:/data
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - /opt/appdata/sonarr:/config
      - /data:/data
    restart: unless-stopped

  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - /opt/appdata/lidarr:/config
      - /data:/data
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - /opt/appdata/prowlarr:/config
    restart: unless-stopped

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    network_mode: "service:gluetun"
    environment:
      - LOG_LEVEL=info
    restart: unless-stopped
```
Make sure to replace the `YOUR_NORDVPN_KEY_HERE` by the actual key (the 2nd one) we generated earlier.

To start docker, run:
```bash
sudo docker compose -f servarr-compose.yml up -d
```

### Outside VPN vault
Here, we set up the media delivery part of the stack which can be outside the vault. We name the docker `yml` file for this part `media-compose.yml`. Create this file and paste the following contents inside:
```bash
version: "3.8"
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - /opt/appdata/jellyfin:/config
      - /data/media:/data/media
    ports:
      - 8096:8096
    restart: unless-stopped

  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    environment:
      - ND_SCANSCHEDULE=1h
      - ND_LOGLEVEL=info  
    volumes:
      - /opt/appdata/navidrome:/data
      - /data/media/music:/music
    ports:
      - 4533:4533
    restart: unless-stopped

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - /opt/appdata/jellyseerr:/app/config
    ports:
      - 5055:5055
    restart: unless-stopped
```

Same as before, to start the container, run:
```bash
sudo docker compose -f servarr-compose.yml up -d
```

## External routing (optional)
It is convenient (but completely optional) to able to access these services through a domain instead of having to tunnel into your local network (tho of course, the latter is more secure). To make sure the external routing is secure, we will not reinvent the wheel and simply use [Cloudlfare](https://www.cloudflare.com/) zero-trust free service. Note, that Cloudlfare UI is bound to change so in this guide the paths described correspond to the semantics of where you want to go rather to actual buttons on Cloudlfare's webpage.


Go to Cloudlfare and navigate to `Zero Trust > Networks > Tunnels`. Create a CloudeFlared tunnel and run the command that should be present at the end of the setup in your local terminal (on the server that these services will be running on).
Next, we need to create mappings for the local ports. With the default ports, this would be something like:
- jellyfin.yourdomain.com --> http://localhost:8096
- music.yourdomain.com --> http://localhost:4553
- radarr.yourdomain.com --> http://localhost:7878
- sonarr.yourdomain.com --> http://localhost:8989
- requests.yourdomain.com --> http://localhost:5055
- lidarr.yourdomain.com --> http://localhost:8686

Since we are exposing ports to the internet, we want to make sure this is secure by using Cloudlfare's zero-trust "lockdown". We can set it up by:
- going to `Access > Applications`
- Add a "Self-Hosted Application" and name it (e.g. Servarr Protect)
- Apply it to the "radarr", "sonarr", and "lidarr" subdomains.
- Create and apply a policy (in the cloudflare UI) allowing only a personal email address via a PIN code. Note that Jellyfin, Jellyseer and Navidrome should be left open.

## Configuration(s)
### qBittorrent
Open `http://localhost:8080` on the computer running the stack (i.e. the server). The UI will ask you to login, the temporary password can be access by running:
```bash
sudo docker logs qBittorrent
```
It is probably a good idea to change the temporary password via the UI (settings).

After logging in, we need to make sure that the downloads folder is the one we set up earlier. Navigate to `Options > Downloads` in the qBittorrent UI, and change the default safe path to `/data/torrents`.

To make sure the network "stuff" is correctly connected and the killswitch (turn off internet if something goes down to avoid IP exposure etc.) is working, go to `Options > Advanced` and change the network interface to "tun0". Save and restart:
```bash
sudo docker restart qbittorrent
```

To make the setup easier, it is recommended to bypass authentication for cients on localhost so the different services can talk to each other (locally) without a hassle. To do this, go to `Options > Web UI` and check "Bypass authentication for client on localhost".

(Optionally, you might want to set some rules to avoid seeding forever. The defaults are pretty good, e.g.: "When ratio reaches 1.0, then stop torrent").

### Prowlarr & FlareSolverr
Open `http://localhost:9696`.
To add the FlareSolverr proxy (and avoid being flagged as a bot), go to `Settings > Indexer Proxies`, and add FlareSolverr (`http://localhost:8191`). Give it the tag "flare" and save.

Next, we need to add some trackers. Go to `Indexers` and pick some popular ones from the list (e.g. 1337x or piratebay). Make sure to assign the tag "flare" from the previous step. Save.

Next, we link the apps: Radarr (`http://localhost:7878`), Sonarr (`http://localhost:8989`), and Lidarr (`http://localhost:8686`). You will need to get the API keys from the settings of each of the respective apps. Test. Save. Sync.

### Radarr, Sonarr, Lidarr
The configuration for these three is pretty much equivalent and boils down to:
- Set the root folder: `Settings > Media Management`. Add `/data/media/movies` (radarr), `/data/media/tv` (sonarr), or `/data/media/music` (lidarr).
- Connect qBittorrent: `Settings > Download Clients`. Add qBittorrent (`localhost:8080`).

### Jellyseer
(This part is optional, using Jellyfin by itself is perfectly plausbile ... but Jellyseer gives more of that "using Netflix" vibe.)

With Jellyseer properly setup (which ideally will be the case after following this section of this guide), it can serve as the single access point for all of the things we set up earlier. We have mapped Jellyseer to `requests.yourdomain.com`. 

First, we need to connect the backend(s):
- Jellyfin: Sign-in with your Jellyfind admin credentials and sync the libraries.
- Radar/Sonarr: Jellyseer is not in the "VPN stack", meaning to access these from the Jellyseer, we must use the server's local LAN ip (e.g. `192.168.1.10` - you can find this out by running `hostname -I` in the terminal). Then, the full address for radarr would be `192.168.1.10:7878` and for sonarr `192.168.1.10:8989`. 
(Note, that Jellyseer does not support Lidarr. To access Lidar through the browser, one can simply go to `music.yourdomain.com`. Alternatively, to access it in the phone, [Symfonium](https://symfonium.app/) is a good option. For a Spotify-like experience in the browser, see [feishin](https://github.com/jeffvli/feishin/blob/development/media/preview_album_detail.png).)

## Fixes & Tips
### The "Play" button on Jellyseer
To make that button work correctly and actually redirect to Jellyfin (where we will actually consume the media), go to `Settings > Jellyfin` in Jellyseer's GUI. Here, we need to edit the server, i.e. set the external URL to `http://jellyfin.martin-opat.com`.
