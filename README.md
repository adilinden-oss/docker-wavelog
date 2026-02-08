# Wavelog Stack

This document describes the installation of my instance of a [Wavelog](https://www.wavelog.org) VM on Proxmox. This is a containerized installation using Proxmox. Access is provided via logal LAN or Tailscale. Notable features:

- HTTPS using Caddy reverse proxy, custom built to support Cloudflare DNS for certificates
- Tailscale container
- Auxiliary or supplementary container to manage docker volumes

Documentation sites

- https://www.wavelog.org
- https://github.com/wavelog/wavelog
- https://github.com/wavelog/wavelog/wiki/Installation-via-Docker

## System Preparation

Deploy the VM from Debian 13 ISO.

| Parameter  | Value
| ---------- | ----
| BIOS       | SeaBIOS
| Display    | Default
| Machine    | i440fx
| SCSI Ctrl  | VirtIO SCSI single
| QEMU Agent | Yes
| Root Disk  | 32GB
| Memory     | 4096MB
| Cores      | 2
| Username   | logger

## First Login

Perform these steps as `root` user or prepend `sudo` as appropriate. Personally, nearly 100% of CLI activities performed on server/service VM require root, I find connecting as root (using ssh keys, not password) or escalating to root using either `su -` or `sudo /bin/bash -l` is less hassle than dealing with `sudo` for every command.

Configure `/etc/network/interfaces` if it has not been already configured during installation.

    # The primary network interface
    allow-hotplug ens18
    iface ens18 inet static
        address 192.168.1.53/24
        gateway 192.168.1.1

Configure `/etc/resolv.conf`

    nameserver 192.168.1.1

Add `ssh` keys to root and appropriate users.

    mkdir ~/.ssh
    echo '<content of id_ed25519.pub>' >> ~/.ssh/authorized_keys

Install needed packages.

    apt update && apt upgrade -y
    apt install --no-install-recommends -y vim less git qemu-guest-agent curl

## Fancy `motd`

Clone the repo and run the script.

    apt install figlet bsdmainutils
    git clone https://github.com/adilinden-oss/motd.git
    cd motd
    ./install-docker.sh

## Docker

Using the official installation instructions [Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/) and additional hints from [ADS-B Reception, Decoding & Sharing with Docker](https://sdr-enthusiasts.gitbook.io/ads-b/).

Fetch and install `docker`

```
apt install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

```
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

```
apt update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Configure `docker` log rotation by adding this to `/etc/docker/daemon.json`

    {
      "log-driver": "local",
      "log-opts": {
        "max-size": "10m",
        "max-file": "3"
      }
    }

Followed by a restart of the daemon

    service docker restart
    service docker status

Test that `docker compose` is installed correctly

    docker compose version

## Wavelog

### Prepare

My DNS zone is in Docker. The reverse proxy is Caddy. Unfortunately there is no official Caddy image that includes certificate creating using DNS challenges. This environment includes a `Dockerfile` to create a Caddy container that includes support for DNS challenge via CLoudflare DNS.

The Caddy container requires a Cloudflare API token. Find API Token in your Cloudflare account under Profile.

Create an Custom API token with oermissions:

- Zone:Zone:Read
- Zone:DNS:Edit

Zone Resources

- include:specific zone:[choose a domain]

Client IP Address Filtering

- Define a IP or subnet to limit from where the API Token can be used

Make note of the API Token when it is displayed. I am not sure if it can be looked up again.

### Installation

Installing the git repo.

    mkdir /opt
    cd /opt
    git clone https://github.com/adilinden-oss/docker-wavelog.git
    cd docker-wavelog

Edit `.env` with your specific information. I like to lock down containers with specific versions, rather than `latest`.

Bring up the stack

    docker compose build
    docker builder prune -f
    docker compose up -d

Now look at the log and find the URL the `wavelog-tailscale` container spit out. Visit that URL to enrol the container into your Tailscale account. Tailscale permissions and settings are left to the reader. Do make note of the full name for the `log.[your tailnet name].ts.net` machine.

Use the `wavelog-aux` container to modify Wavelog configuration.

### Set the `base_url`

This function will set the `base_url` for Wavelog dynamically, based on a list of whitelisted domains.

Open `/var/www/html/application/config/docker/config.php` in the `wavelog-main` container, or `wavelog/config/config.php` in the `wavelog-aux` container. Comment the existing `$config['base_url']` and replace it with the function below. Adjust the example domains with the desired domain or IP. Do not include a port number, it is dynamically detected and added to the generated `base_url` as needed.

To use the `wavelog-aux` container

- Stop the containers    

      docker compose down

- Access the `wavelog-aux` container    

      docker compose run --rm wavelog-aux

- At the bash prompt install an editor of choice, I prefer `vim`, but `nano` is also popular    

      apt update
      apt install vim nano

- Edit the config file `wavelog/config/config.php`
- Find `$config['base_url']` in the file and replace with    

  ```
  $config['base_url'] = call_user_func(function() {
      $url_allowed = ['example.com', '192.168.1.53', 'log.great-owl.ts.net'];
      $url_stripped = preg_replace('/:\d+$/', '', $_SERVER['HTTP_HOST']);
  
      if (in_array($url_stripped, $url_allowed)) {
          $is_https = (!empty($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https')
                   || (!empty($_SERVER['HTTP_X_FORWARDED_SSL']) && $_SERVER['HTTP_X_FORWARDED_SSL'] === 'on')
                   || (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on')
                   || (!empty($_SERVER['SERVER_PORT']) && $_SERVER['SERVER_PORT'] == 443);
          
          return ($is_https ? "https" : "http") . "://{$_SERVER['HTTP_HOST']}/";
      } else {
          return 'https://example.com/';
      }
  });
  ```

- Edit the `$url_allowed` array with your specific domain or IP addresses. Omit any ports as they will be dynamically added as needed
- Save the changes and exit the container
- Bring up the containet stack    

      docker compose up -d

- Access Wavelog via its various URL

