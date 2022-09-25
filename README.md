# Homescripts README

This repo was forked from [Animosity 22 repository](https://github.com/animosity22/homescripts).

This is my configuration that works for me the best for my use case of an all in one media server. This is not meant to be a tutorial by any means as it does require some knowledge to get setup.

I use the latest rclone stable version downloaded direclty via the [script install](https://rclone.org/install/#script-installation) as package managers are frequently out of date and not maintained. I see no need to use docker for my rclone setup as it's a single binary and easier to maintain and use without being in a docker.

[Change Log](https://github.com/RDeluxe/homescripts/blob/master/Changes.MD)

## Home Configuration

- And old PC
- OneDrive Business for cloud
- Ubuntu 22.04
- Intel(R) Core(TM) i5-3570k CPU @ 3.40GHz
- 16 GB of Memory
- 250GB - SSD root/system disk
- Spinning disk for big storage pools

## Services

- Roon server installed via their nice installer
- Docker
- rclone (native)
- Plex (on docker)
- Deluge (on docker)
- Steam link

# Cloud and media configuration

## My Workflow

The design goals for the workflow are to limit the amount of applications that being used and limit the amount of scripts that are being used as part of the process. That was the reason to remove mergerfs and the upload script from the workflow. This does remove the ability to use hard links in the process but the trade off of having duplicated files for a short period outweighed the con.

Worfklow Pattern (TBD):

1. Sonarr/Radarr identify a file to be downloaded
2. qBit/NZBget downloads a file to local spinning disk (/data)
3. Sonarr/Radarr see the download is complete, file is copied from spinning disk (/data) to the respective rclone mount (/media/Movies or media/TV)
4. Rclone waits the delay time (1 hour in this setup) and uploads the file to the remote

This workflow has a lot less moving parts and reduces the amount of things that can break. There is a local cache drive for rclone (/cache) that is used for the vfs-cache-mode full that stores the uploads before they get uploaded and any downloaded cache files. The only breakpoint here is if the cache area gets full, but generally that should not happen as files are uploaded within an hour and it is a 2TB SSD in this setup that should offer plenty of space. If that disk is too small, there can be issue with it filling up and creating an issue. Disk is cheap enough though that should not be a problem.

### Installation

My Linux setup:

```bash
lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 22.04 LTS
Release:	22.04
Codename:	jammy
```

Fuse needs to be installed for a rclone mount to function. `allow-other` is for my use and not recommended for shared servers as it allows any user to see a rclone mount. I am on a dedicated server that only I use so that is why I uncomment it.

You need to make the change to /etc/fuse.conf to `allow_other` by uncommenting the last line or removing the # from the last line.

```bash
sudo vi /etc/fuse.conf
root@gemini:~# cat /etc/fuse.conf
# /etc/fuse.conf - Configuration file for Filesystem in Userspace (FUSE)

# Set the maximum number of FUSE mounts allowed to non-root users.
# The default is 1000.
#mount_max = 1000

# Allow non-root users to specify the allow_other or allow_root mount options.
user_allow_other

```

Two rclone mounts are used and this could be expanded if there was a need for multiple mount points. The goal here is each mount point for unique for each "pair" of applications uses. As an example, Sonarr/Plex use the /media/films mount and point to that specifically. This allows for downloads and uploads to work on a mount point. Any uploads are handled by rclone without the need for an additional upload script. The upload delay is configurable and 1 hour is the parameter being used here.

The "films" folder is badly named, it stores both movies and shows.

```bash
/media/films (rclone mount with vfs cache mode full)
/media/music (rclone mount with vfs cache mode full)
```

My `rclone.conf` has an only one entry, for my OneDrive account.

My rclone looks like: [rclone.conf](https://github.com/RDeluxe/homescripts/blob/master/rclone.conf)

They are all mounted via systemd scripts. rclone is mounted first followed by the mergerfs mount.

My media starts up items in order:

1. [rclone-films service](https://github.com/RDeluxe/homescripts/blob/master/systemd/rclone-films.service) This is a standard rclone mount, the post execution command allows for the caching of the file structure in a single systemd file that simplies the process.
2. [rclone-music service](https://github.com/RDeluxe/homescripts/blob/master/systemd/rclone-music.service) This is a standard rclone mount, the post execution command allows for the caching of the file structure in a single systemd file that simplies the process.

### Docker

With the exception of rclone, all applications are setup in a docker-compose and leverage docker for ease of use, maintenance and upgrades. With Plex, it is advised to leverage a docker as for ensuring that hardware transcoding and HDR tone mapping support, only certain Linux OS flavors work easy without docker. By putting Plex in a docker, there is minimal configuration that needs to be done to get full hardware support as depending on hardware, it does get complex.

[Plex HDR Tone Mapping Support](https://support.plex.tv/articles/hdr-to-sdr-tone-mapping/)

Docker install for each operating system can be instructions are here: [Docker Install Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

The docker-compose.yml below is what is being used for multiple applications as Sonarr, Radarr and Plex are included below. The key for hardware support is ensuring that /dev/dri is mapped and a single UID/GID is consistent in the configuration as UID=1000 and GID=1000 is the only user configured on my single server setup.

The docker setup is configured in /opt/docker and all the data for every application is stored in /opt/docker/data in this configuration. That is backed up on a daily basis to another location and occassinally to cloud storage depending on the risk appetite.

#### My Docker-Compose

The override below forces the docker server to require the rclone mounts and if the rclone mounts stop, systemd will stop all the docker services that are running. This allow the dependencies to be done to ensure that the applications do not lose media and stop if an issue with rclone occurs.

```bash
gemini:/etc/systemd/system/docker.service.d # cat override.conf
[Unit]
After=rclone-movies.service rclone-tv.service
Requires=rclone-movies.service rclone-tv.service
```

I use docker compose for all my services and have portainer there for easier looking at things when I don't want to connect to a console. I use the same user ID/groups for my docker to simpify permissions. My plex compose is basic and looks like:

```bash
  plex:
    image: lscr.io/linuxserver/plex
    container_name: plex
    network_mode: host
    devices:
     - /dev/dri:/dev/dri
    privileged: true
    environment:
      - PUID=1000
      - PGID=1000
      - VERSION=docker
      - TZ=America/New_York
    volumes:
      - /opt/docker/data/plex:/config
      - /media/films/Movies:/media/Movies
      - /media/films/Shows:/media/TV
    restart: unless-stopped
```

/dev/dri is a must for hardware transocding.

## Plex Tweaks

This is a legacy tweak for the plexmediaserver service to get around it running as the plex user and require services be running. This allows me to keep my trash empty on as if mount has a problem, it will stop plex. This requires /var/lib/plexmediaserver (on Linux) to be changed to the owner and group defined below.

```bash
gemini: /etc/systemd/system/plexmediaserver.service.d # cat override.conf
[Unit]
After=rclone-movies.service rclone-tv.service
Requires=rclone-movies.service rclone-tv.service

[Service]
User=felix
Group=felix
```

These tips and more for Linux can be found at the [Plex Forum Linux Tips](https://forums.plex.tv/t/linux-tips/276247)

### Plex

- `Enable Thumbnail previews` - off: This creates a full read of the file to generate the preview and is set per library that is setup
- `Perform extensive media analysis during maintenance` - off: This is listed under Scheduled Tasks and does a full download of files and is ony used for bandwidth analysis when streaming.

### Sonarr/Radarr

- `Analyze video files` - off: This also fully downloads files to perform analysis and should be turned off as this happens frequently on library refreshes if left on.

## Caddy Proxy Server

I use Caddy to server majority of my things as I plug directly into GitHub oAuth2 for authentication. I can toggle CDN on and off via the proxy in the DNS.

My configuration is [here](https://github.com/RDeluxe/homescripts/blob/master/PROXY.MD).

## TODO

- adjust my mounts on my Linux machine to use BTRFS over EXT4/XFS and found a huge performance improvement.
