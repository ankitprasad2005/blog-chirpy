---
title: Setting up *ARR stack
author: ankitprasad2005
layoyt: post
date: 2024-07-04 14:10:00 +0530
categories: [Self Hosted, Arr Stack]
tags: [Guide, Arr Stack, Media Server, Self Hosted, DevOps]
render_with_liquid: true
image: assets/blog/arr_stack/arr-1.png
---


Guide to setup *ARR stack - self hosted streaming platform
====
Arr is a bunch of services made by [servarr](https://wiki.servarr.com/) org. The services basically allows to easily add movies / TV shoes to your media server. Not just that here i have gived some services which will fetch the movie or series you want put it to download. When download is completed it adds it to the media server and download the caption for it all automatically.

## Index

- [Guide to setup \*ARR stack - self hosted streaming platform](#guide-to-setup-arr-stack---self-hosted-streaming-platform)
  - [Index](#index)
  - [File Structure](#file-structure)
  - [Arr Services](#arr-services)
    - [Radarr](#radarr)
    - [Sonarr](#sonarr)
  - [Arr indexer manager](#arr-indexer-manager)
    - [Prowlarr](#prowlarr)
    - [Flaresolver](#flaresolver)
    - [Jackett](#jackett)
  - [Download manager](#download-manager)
    - [Qbittorrent](#qbittorrent)
  - [Media Playback Server](#media-playback-server)
    - [Plex](#plex)
    - [Emby](#emby)
    - [Jellyfin](#jellyfin)
  - [Other usefull services](#other-usefull-services)
    - [Bazarr](#bazarr)
    - [Unpackerr](#unpackerr)
    - [Watchstate](#watchstate)
    - [Overseerr](#overseerr)
  - [VPN](#vpn)
    - [Gluetun](#gluetun)
  - [Combined Docker Compose](#combined-docker-compose)
  - [How to run](#how-to-run)
  - [Post installation](#post-installation)
- [Sources](#sources)


## File Structure
For storing all the config file for each service that we are using
```
config
├── bazarr
├── emby
├── gluetun
├── jackett
├── jellyfin
├── plex
├── prowlarr
├── qbittorrent
├── radarr
├── sonarr
├── unpackerr
└── watchstate
```

For storing all the media file,    
The downloading media will be there in the torrents or usenet (whatever you are using).    
The completed media will be moved in the media directory by readarr or sonarr.    

```
data
├── incomplete
│   ├── movies
│   └── tv
├── media
│   ├── books
│   ├── movies
│   └── tv
├── torrents
│   ├── books
│   ├── incomplete
│   ├── movies
│   └── tv
└── usenet
    ├── books
    ├── incomplete
    ├── movies
    └── tv
```


## Arr Services
### Radarr   
Its use to add, manage and remove movies. It searches the queried movie to the provided indexer and starts download via the download manager and moniters till its over downloading.

```yaml
radarr:
  image: linuxserver/radarr
  user: ${UID}:${GID}
  restart: always
  ports:
    - "${PORT_RADARR}:7878"
  environment:
    - PGID=${GID}
    - PUID=${UID}
    - TZ=${TIMEZONE:?err}
  volumes:
    - ${CONFIG}/radarr:/config
    - ${DATA}:/data
```

### Sonarr    
Its use to add, manage and remove tv shows (and web series). It searches the queried show or series to the provided indexer and starts download via the download manager and moniters till its over downloading.

 ```yaml
sonarr:
   image: linuxserver/sonarr
   user: ${UID}:${GID}
   restart: always
   ports:
     - "${PORT_SONARR}:8989"
   environment:
     - PGID=${GID}
     - PUID=${UID}
     - TZ=${TIMEZONE:?err}
   volumes:
     - ${CONFIG}/sonarr:/config
     - ${DATA}:/data
```

## Arr indexer manager
### Prowlarr    
This service allows you to add indexer to Radarr and Sonarr.

```yaml
prowlarr:
  image: linuxserver/prowlarr
  user: ${UID}:${GID}
  restart: always
  ports:
    - "${PORT_PROWLARR}:9696"
  cap_add:
    - NET_ADMIN
  environment:
    - PGID=${GID}
    - PUID=${UID}
    - TZ=${TIMEZONE:?err}
  volumes:
    - ${CONFIG}/prowlarr:/config
    - ${DATA}:/data
```

### Flaresolver    
This is used to solve the cloudflare capcha. (currently not functional)

```yaml
flaresolverr:
  image: ghcr.io/flaresolverr/flaresolverr:latest
  user: ${UID}:${GID}
  restart: unless-stopped
  ports:
    - "${PORT_FLARESOL}:8191"
  environment:
    - PGID=${GID}
    - PUID=${UID}
    - LOG_LEVEL=debug
    - LOG_HTML=true
    # Enables hcaptcha-solver => https://github.com/JimmyLaurent/hcaptcha-solver
    - CAPTCHA_SOLVER=hcaptcha-solver
    # Enables CaptchaHarvester => https://github.com/NoahCardoza/CaptchaHarvester
    # - CAPTCHA_SOLVER=harvester
    # - HARVESTER_ENDPOINT=https://127.0.0.1:5000/token
    - TZ=${TIMEZONE:?err}
```

### Jackett
This is another service that lets you add idexer but you have to import it manually to redarr and sonarr.

```yaml
jackett:
    image: lscr.io/linuxserver/jackett:latest
    user: ${UID}:${GID}
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=${TIMEZONE:?err}
      #- AUTO_UPDATE=true #optional
      #- RUN_OPTS= #optional
    volumes:
      - ${CONFIG}/jackett:/config
      - ${DATA}:/data
    ports:
      - ${PORT_JACKETT}:9117
    restart: unless-stopped
```

## Download manager
### Qbittorrent    
A torrent download manager. Let you download the magnet link and the .torrent file given my the arr services.

```yaml
qbittorrent:
  image: linuxserver/qbittorrent
  user: ${UID}:${GID}
  restart: unless-stopped
  volumes:
    - ${CONFIG}/qbittorrent:/config
    - ${DATA}:/data
  environment:
    - PUID=${PUID}
    - PGID=${GUID}
    - TZ=${TIMEZONE:?err}
    - WEBUI_PORT=${PORT_QBIT}
  ports:
    - "${PORT_QBIT}:${PORT_QBIT}"
    - 6881:6881 # BitTorrent
    - 6881:6881/udp
```

## Media Playback Server
### Plex     
A web server for streaming the media that you downloaded. The mobile client of it and transcoding is paid. 

```yaml
plex:
  image: plexinc/pms-docker:latest # Set specific: plexinc/pms-docker:1.40.2.8395-c67dce28e
  ports:
    - ${PORT_PLEX}:32400/tcp
    - 8324:8324/tcp
    - 32469:32469/tcp
    - 1900:1900/udp
    - 32410:32410/udp
    - 32412:32412/udp
    - 32413:32413/udp
    - 32414:32414/udp
  environment:
    - PUID=${PUID}
    - PGID=${GUID}
    - TZ=${TIMEZONE:?err} # Change this to match your server's timezone
    #- PLEX_CLAIM=${CLAIM_KEY}
    - ADVERTISE_IP="${ADV_IP}"
  hostname: ${HOSTNAME}
  volumes:
    - ${CONFIG}/plex:/config
    #- ${DATA}/media:/media
    - ${DATA}:/data
  restart: unless-stopped
```

### Emby    
Another web server for streaming media which was previously open source but no longer so. Its mostly free but some features like download in android is paid. 
	
```yaml
emby:
  image: emby/embyserver
  hostname: ${HOSTNAME}
  # runtime: nvidia # Expose NVIDIA GPUs
  # network_mode: host # Enable DLNA and Wake-on-Lan
  environment:
    - UID=${PUID} # The UID to run emby as (default: 2)
    - GID=${PGID} # The GID to run emby as (default 2)
  volumes:
    - ${CONFIG}/emby:/config # Configuration directory
    # - ${DATA}/media:/media
    - ${DATA}:/data
  ports:
    - ${PORT_EMBY}:8096 # HTTP port
  # devices:
  #   - /dev/dri:/dev/dri # VAAPI/NVDEC/NVENC render nodes
  restart: always
```

### Jellyfin    
Its another one of the web server just the difference being this one is a open source one ironically build over the last open source version of emby but is still maintained by its own community.

```yaml
jellyfin:
  image: jellyfin/jellyfin
  user: ${UID}:${GID}
  volumes:
    - ${CONFIG}/jellyfin:/config
    - ${DATA}:/data
  ports:
    - ${PORT_JELLYFIN}:8096
  environment:
    - TZ=${TIMEZONE:?err}
  restart: unless-stopped
``` 

## Other usefull services
### Bazarr    
This is a service that monitors the movies and tv shows for their subtitles. If it dosnt have a subtitle it just fetches it from the source you set like open-subtitles, etc.

 ```yaml
bazarr:
  image: ghcr.io/hotio/bazarr:latest
  restart: unless-stopped
  logging:
    driver: json-file
  ports:
    - "${PORT_BAZARR}:6767"
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=${TIMEZONE:?err}
  volumes:
    - ${CONFIG}/bazarr:/config
    - ${DATA}:/data
```

### Unpackerr    
It helps to reformat, unzip and other file tasks on files which is not directly usable or not in the usable structure.

```yaml
unpackerr:
  image: golift/unpackerr
  volumes:
    - ${DATA}/torrents:/downloads
    - ${CONFIG}/unpackerr:/config
  environment:
    - TZ=${TIMEZONE:?err}
    - UN_FOLDER_0_PATH=/downloads
    - UN_LOG_FILE=/config/unpackerr.log
    - UN_SONARR_0_URL=http://sonarr:8989
    - UN_SONARR_0_API_KEY=${API_SONARR}
    - UN_RADARR_0_URL=http://radarr:7878
    - UN_RADARR_0_API_KEY=${API_RADARR}
  restart: unless-stopped
```

### Watchstate    
It help sync watch state between plex, emby and jellyfin so if you are watching a show or movie in one of the platform and switch to another one you watch state will be updated there too.

```yaml
watchstate:
  image: ghcr.io/arabcoders/watchstate:latest
  user: ${UID}:${GID}
  ports:
    - "${PORT_WATCHSTATE}:8080" # The port which will serve WebUI + API + Webhooks
  volumes:
    - ${CONFIG}/watchstate:/config # mount current directory to container /config directory.
  restart: unless-stopped
```

### Overseerr
Its a server where you can add movies which you need to be added in your service. You can even share access to others and aprove their request.

```yaml
overseerr:
  image: "sctx/overseerr:latest"
  environment:
    - PGID=${GID}
    - PUID=${UID}
    - LOG_LEVEL=debug
    - TZ=${TIMEZONE:?err}
  ports:
    - "${PORT_OVERSEERR}:5055"
  volumes:
    - ${DATA}/overseerr:/app/config
    - ${MEDIA}/tv:/mnt/tv
    - ${MEDIA}/movies:/mnt/movies
    - ${MEDIA}/downloads:/downloads
  restart: unless-stopped
```

## VPN
### Gluetun
This is a VPN client that connects to your preferred VPN provider. Go to Gluetuns guide for configuring it.


## Combined Docker Compose

```yaml
services:
  emby:
    image: emby/embyserver
    hostname: ${HOSTNAME}
    # runtime: nvidia # Expose NVIDIA GPUs
    # network_mode: host # Enable DLNA and Wake-on-Lan
    environment:
      - UID=${PUID} # The UID to run emby as (default: 2)
      - GID=${PGID} # The GID to run emby as (default 2)
    volumes:
      - ${CONFIG}/emby:/config # Configuration directory
      # - ${DATA}/media:/media
      - ${DATA}:/data
    ports:
      - ${PORT_EMBY}:8096 # HTTP port
    # devices:
    #   - /dev/dri:/dev/dri # VAAPI/NVDEC/NVENC render nodes
    restart: always

  qbittorrent:
    image: linuxserver/qbittorrent
    user: ${UID}:${GID}
    restart: unless-stopped
    volumes:
      - ${CONFIG}/qbittorrent:/config
      - ${DATA}:/data
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TIMEZONE:?err}
      - WEBUI_PORT=${PORT_QBIT}
    ports:
      - "${PORT_QBIT}:${PORT_QBIT}"
      - 6881:6881 # BitTorrent
      - 6881:6881/udp

  radarr:
    image: linuxserver/radarr
    user: ${UID}:${GID}
    restart: always
    ports:
      - "${PORT_RADARR}:7878"
    environment:
      - PGID=${GID}
      - PUID=${UID}
      - TZ=${TIMEZONE:?err}
    volumes:
      - ${CONFIG}/radarr:/config
      - ${DATA}:/data

  sonarr:
    image: linuxserver/sonarr
    user: ${UID}:${GID}
    restart: always
    ports:
      - "${PORT_SONARR}:8989"
    environment:
      - PGID=${GID}
      - PUID=${UID}
      - TZ=${TIMEZONE:?err}
    volumes:
      - ${CONFIG}/sonarr:/config
      - ${DATA}:/data

  bazarr:
    image: ghcr.io/hotio/bazarr:latest
    restart: unless-stopped
    logging:
      driver: json-file
    ports:
      - "${PORT_BAZARR}:6767"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=${TIMEZONE:?err}
    volumes:
      - ${CONFIG}/bazarr:/config
      - ${DATA}:/data

  prowlarr:
    image: linuxserver/prowlarr
    user: ${UID}:${GID}
    restart: always
    ports:
      - "${PORT_PROWLARR}:9696"
    cap_add:
      - NET_ADMIN
    environment:
      - PGID=${GID}
      - PUID=${UID}
      - TZ=${TIMEZONE:?err}
    volumes:
      - ${CONFIG}/prowlarr:/config
      - ${DATA}:/data

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    user: ${UID}:${GID}
    restart: unless-stopped
    ports:
      - "${PORT_FLARESOL}:8191"
    environment:
      - PGID=${GID}
      - PUID=${UID}
      - LOG_LEVEL=debug
      - LOG_HTML=true
      # Enables hcaptcha-solver => https://github.com/JimmyLaurent/hcaptcha-solver
      - CAPTCHA_SOLVER=hcaptcha-solver
      # Enables CaptchaHarvester => https://github.com/NoahCardoza/CaptchaHarvester
      # - CAPTCHA_SOLVER=harvester
      # - HARVESTER_ENDPOINT=https://127.0.0.1:5000/token
      - TZ=${TIMEZONE:?err}

  jackett:
    image: lscr.io/linuxserver/jackett:latest
    user: ${UID}:${GID}
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=${TIMEZONE:?err}
      #- AUTO_UPDATE=true #optional
      #- RUN_OPTS= #optional
    volumes:
      - ${CONFIG}/jackett:/config
      - ${DATA}:/data
    ports:
      - ${PORT_JACKETT}:9117
    restart: unless-stopped

  overseerr:
    image: "sctx/overseerr:latest"
    environment:
      - PGID=${GID}
      - PUID=${UID}
      - LOG_LEVEL=debug
      - TZ=${TIMEZONE:?err}
    ports:
      - "${PORT_OVERSEERR}:5055"
    volumes:
      - ${DATA}/overseerr:/app/config
      - ${MEDIA}/tv:/mnt/tv
      - ${MEDIA}/movies:/mnt/movies
      - ${MEDIA}/downloads:/downloads
    restart: unless-stopped

  plex:
    image: plexinc/pms-docker:latest # Set specific: plexinc/pms-docker:1.40.2.8395-c67dce28e
    ports:
      - ${PORT_PLEX}:32400/tcp
      - 8324:8324/tcp
      - 32469:32469/tcp
      - 1900:1900/udp
      - 32410:32410/udp
      - 32412:32412/udp
      - 32413:32413/udp
      - 32414:32414/udp
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TIMEZONE:?err} # Change this to match your server's timezone
      #- PLEX_CLAIM=${CLAIM_KEY}
      - ADVERTISE_IP="${ADV_IP}"
    hostname: ${HOSTNAME}
    volumes:
      - ${CONFIG}/plex:/config
      #- ${DATA}/media:/media
      - ${DATA}:/data
    restart: unless-stopped

  jellyfin:
    image: jellyfin/jellyfin
    user: ${UID}:${GID}
    volumes:
      - ${CONFIG}/jellyfin:/config
      - ${DATA}:/data
    ports:
      - ${PORT_JELLYFIN}:8096
    environment:
      - TZ=${TIMEZONE:?err}
    restart: unless-stopped

  unpackerr:
    image: golift/unpackerr
    volumes:
      - ${DATA}/torrents:/downloads
      - ${CONFIG}/unpackerr:/config
    environment:
      - TZ=${TIMEZONE:?err}
      - UN_FOLDER_0_PATH=/downloads
      - UN_LOG_FILE=/config/unpackerr.log
      - UN_SONARR_0_URL=http://sonarr:8989
      - UN_SONARR_0_API_KEY=${API_SONARR}
      - UN_RADARR_0_URL=http://radarr:7878
      - UN_RADARR_0_API_KEY=${API_RADARR}
    restart: unless-stopped

  watchstate:
    image: ghcr.io/arabcoders/watchstate:latest
    user: ${UID}:${GID}
    ports:
      - "${PORT_WATCHSTATE}:8080" # The port which will serve WebUI + API + Webhooks
    volumes:
      - ${CONFIG}/watchstate:/config # mount current directory to container /config directory.
    restart: unless-stopped
```

A sample `.env` file for the given compose

```
# basic system details
TIMEZONE=Asia/Kolkata
HOSTNAME=

# if you want to run application with specific user access
UID=1000
GID=1000

# path to storage
DATA=/path/to/media
MEDIA=/path/to/config

# all ports that you can expose
PORT_EMBY=
PORT_PLEX=
PORT_FLARESOL=
PORT_PROWLARR=
PORT_RADARR=
PORT_SONARR=
PORT_FLOOD=
PORT_QBIT=
PORT_OVERSEERR=

# API keys (you can keep this empty for the first run)
API_RADARR=
API_SONARR=
```

## How to run
```bash
docker compose up -d
```

## Post installation
1. Login to all the services via the web port and setup.
2. Get the API key for radarr and sonarr from settings.
3. All that to the .env and restart the docker compose.

# Sources
1. [trash-guides](https://trash-guides.info/)
2. [WikiArr](https://wiki.servarr.com/)

