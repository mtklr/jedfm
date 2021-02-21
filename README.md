# jedfm (just enough docker for me)

A simple docker-compose helper script. All it really does is combine `.yml` files into a `docker-compose` file.

I'm using a Raspberry Pi 4 with latest Raspbian installed.

## setup

Install `docker-ce` per your system.

```console
sudo apt install docker-ce

# everything else happens here
cd jedfm
```

Create `servicename.yml` files in `./services`. Note that these files start with the name indented two spaces in.

Example: `./services/pihole.yml`:

```yaml
  pihole:
    image: pihole/pihole
    container_name: pihole
    restart: unless-stopped
    # https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    cap_add:
      - NET_ADMIN # Recommended but not required (DHCP needs NET_ADMIN)
    environment:
      - PUID=$DOCK_UID
      - PGID=$DOCK_GID
      - TZ=$DOCK_TZ
      - ADMIN_EMAIL=$DOCK_MAIL_TO
      - IPv6=false
      - PIHOLE_DNS_=127.0.0.1;1.1.1.1
      # - WEBPASSWORD=changeme # set a secure password here or it will be random
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "80:80/tcp"
      # doesn't really need port 443
      # https://github.com/pi-hole/docker-pi-hole/issues/755
      - "443:443/tcp"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - $DOCK_CONFDIR/pihole/etc-pihole:/etc/pihole
      - $DOCK_CONFDIR/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
```

Copy `./env.example` to `./.env`, and adjust to suit your setup:

```sh
# edit and save as .env
DOCK_HOME="$PWD"
DOCK_SERVICESDIR="$DOCK_HOME/services"

# for me, ./up is a symlink to external storage at /srv/hd/dock/up
DOCK_CONFDIR=./up/config
DOCK_STORAGEDIR=./up/storage

DOCK_COMPOSEFILE=docker-compose.yml
DOCK_COMPOSEVERSION=2.1

DOCK_HOSTNAME=example.org
DOCK_IP=192.168.1.2

DOCK_UID=1000
DOCK_GID=1000

DOCK_MAIL_TO=user@mail.com
DOCK_MAIL_SMTP_HOST=smtp.mail.com
DOCK_MAIL_SMTP_PORT=587

DOCK_PASSLEN=16

# https://blog.packagecloud.io/eng/2017/02/21/set-environment-variable-save-thousands-of-system-calls/
DOCK_TZ=:/etc/localtime
```

### why `DOCK_TZ=:/etc/localtime` in `./.env`?

[Explanation](https://blog.packagecloud.io/eng/2017/02/21/set-environment-variable-save-thousands-of-system-calls/). I imagine this applies to containers as well, haven't tested it.

On the host (the Raspberry Pi), `/etc/environment` contains:

```sh
TZ=:/etc/localtime
```

Add entry to `./services/name.yml` under `volumes:'

```yaml
volumes:
  - /etc/localtime:/etc/localtime:ro
```

Otherwise, set `DOCK_TZ=America/New_York` to the appropriate timezone in `./.env` and comment out or remove this entry under `volumes:`

```yaml
# - /etc/localtime:/etc/localtime:ro in services/*
```

* I `ssh` to the docker host, and just noticed that `TZ` isn't set, even though `/etc/environment` contains `TZ=:/etc/localtime`. This happens to be because I'm connecting to the docker host using `ssh`, and `/etc/ssh/sshd_config` contains `UsePAM no` - I'm using key-based login, and this is a recommended setting. However, `UsePAM yes` restores the environment setup (`TZ` is now set when I connect). This may have other implications, [use what you think is best](https://duckduckgo.com/?t=ffab&q=ssh+usepam&ia=web).

## usage

```console
usage: dock [option] [name ...]
Make a docker-compose.yml file from /home/pi/jedfm/services/*.yml.

options:
  -c         run `docker-compose config` to check docker-compose.yml
  -d [NAME]  disable service NAME. No argument lists disabled services.
  -e [NAME]  enable service NAME. No argument lists enabled services.
  -g [NUM]   generate password NUM length (default: 16)
  -h         show this help and exit
  -l         list services
  -p         list ports defined in services
```

### create `docker-compose.yml` from `./services/*.yml`

Run whenever `docker-compose.yml` should be updated.

```sh

./dock

```

`./dock -c` also runs `docker-container config` to check the configuration.

### start

```console
docker-compose up -d [name]
```

### disable

```console
./dock -d [name ...]
```

No arguments shows what's disabled.

### enable

```console
./dock -e [name ...]
```

No arguments shows what's enabled.

### generate password

```console
./dock -g [length]
```

Just a convenience for services that need new passwords.

### ports

List ports defined in `services/*.yml` (check for duplicate ports in different services):

```console
./dock -p
```

### ctop

[ctop](https://github.com/bcicen/ctop) is a nice, simple way to see what's going on.

```console
sudo apt install docker-ctop

ctop
```

### updates

[diun](https://crazymax.dev/diun/) checks containers for updates.

## aliases

Some helpful `bash` aliases:

```sh
alias d='docker'
alias dcon='docker container'
alias dcc='docker-compose config'
alias dcp='docker-compose'
alias dip="docker ps -q | xargs -n 1 docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} {{ .Name }}' | sed 's/ \// /'"

# completion
complete -F _docker d
complete -F _docker_compose dcp
```

[linuxserver.io tips & tricks](https://docs.linuxserver.io/general/docker-compose#tips-and-tricks)

## configuration notes

### host networking

[docker docs - Use host networking](https://docs.docker.com/network/host/):

> If you use the host network mode for a container, that container’s network stack is not isolated from the Docker host (the container shares the host’s networking namespace), and the container does not get its own IP-address allocated. For instance, if you run a container which binds to port 80 and you use host networking, the container’s application is available on port 80 on the host’s IP address...
>
> Host mode networking can be useful to optimize performance, and in situations where a container needs to handle a large range of ports, as it does not require network address translation (NAT), and no “userland-proxy” is created for each port.

So optionally, in `./services/service.yml`, add `network_mode: host` and comment out `ports:` and related `port` entries.

```yaml
  name:
    network_mode: host
  # ports:
    # - 8080:80
```

### logging

[docker docs - Configure logging drivers](https://docs.docker.com/config/containers/logging/configure/)

> Tip: use the “local” logging driver to prevent disk-exhaustion
>
> By default, no log-rotation is performed. As a result, log-files stored by the default json-file logging driver logging driver can cause a significant amount of disk space to be used for containers that generate much output, which can lead to disk space exhaustion.

> Docker keeps the json-file logging driver (without log-rotation) as a default to remain backward compatibility with older versions of Docker, and for situations where Docker is used as runtime for Kubernetes.

> For other situations, the “local” logging driver is recommended as it performs log-rotation by default, and uses a more efficient file format. Refer to the Configure the default logging driver section below to learn how to configure the “local” logging driver as a default, and the local file logging driver page for more details about the “local” logging driver.

To use local log-driver, create/edit `/etc/docker/daemon.json` on host:

```json
{
  "log-level": "warn",
  "log-driver": "local",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

The above also sets the log level to "warn," and limits the number of log files/file size.

Restart docker service:

```sh
sudo systemctl restart docker.service
```

#### healthcheck logging spam

I recently noticed a lot of entries in `/var/syslog` like this:

```
Feb  7 10:03:28 host systemd[1]: run-docker-runtime\x2drunc-moby-86c2c3f555790d48fea65908bf7a4dca508b5db069125069beacdf98b52513ec-runc.Z8tVA2.mount: Succeeded.
```

A quick search led me to [this github issue](https://github.com/docker/for-linux/issues/679). It seems that containers with healthcheck create these entries. [gertjanklein](https://github.com/docker/for-linux/issues/679#issuecomment-509159724) and [H4R0](https://github.com/docker/for-linux/issues/679#issuecomment-648293209) suggest workarounds:

In short, create `/etc/rsyslog.d/01-blocklist.conf`:

```conf
if $msg contains "run-docker-runtime" and $msg contains ".mount: Succeeded." then {
    stop
}
```

And edit `/etc/systemd/journald.conf`:

```conf
RuntimeMaxuse=10M
```

After, restart these services:

```sh
sudo systemctl restart rsyslog.service
sudo systemctl restart systemd-journald.service
```

I'm using [log2ram](https://github.com/azlux/log2ram) - keep an eye on its usage with `df -lh`.

## advanced config

### reverse proxy

This took me a while to figure out, but pretty nice once it's set up.

Ideally, I wanted to access docker stuff via https. Initially I configured this using [Nginx Proxy Manager](https://nginxproxymanager.com/) and [Cloudflare](https://cloudflare.com), which was fairly straightforward. This involved exposing ports 80 and 443 on my router, though, which I wasn't comfortable with. I don't need to access any of this stuff outside of my home network, and I didn't want to worry about potential vulnerabilities.

So - how to set up a reverse proxy for the local network with https?

#### mkcert

I mostly use a mac laptop at home. This is where I installed [mkcert](https://github.com/FiloSottile/mkcert):

```console
# install mkcert
brew install mkcert nss

# install the local CA in the system trust store
mkcert -install

# generate wildcard cert for local network
mkcert '*.host.home'
```

#### Nginx Proxy Manager

Run [Nginx Proxy Manager](https://nginxproxymanager.com/) via `docker-compose`. `services/nginxpm.yml`:

```yaml
  nginxpm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginxpm
    restart: unless-stopped
    ports:
      # Public HTTP Port:
      - '80:80'
      # Public HTTPS Port:
      - '443:443'
      # Admin Web Port:
      - '81:81'
    environment:
      # If you would rather use Sqlite uncomment this
      # and remove all DB_MYSQL_* lines above
      DB_SQLITE_FILE: "/data/database.sqlite"
      # Uncomment this if IPv6 is not enabled on your host
      DISABLE_IPV6: 'true'
    volumes:
      - $DOCK_CONFDIR/nginxpm/data:/data
      - $DOCK_CONFDIR/nginxpm/letsencrypt:/etc/letsencrypt
```

* Note that compared to the default config, I'm not using a separate database container, just the `.sqlite` file. Seems to work fine. Maybe it's less secure? I don't know.

Log in to Nginx Proxy Manager (NPM), update user, password, etc.

#### Add SSL Certificate

In the NPM interface, under "SSL Certificates," select "Add Certificate" - not the center button, the one above and to the right.

Choose "Custom" from the drop-down list.

Add the key and certificate files (created by `mkcert` above).

#### Add Proxy Host

Adding [Glances](https://github.com/nicolargo/glances) proxy host, as an example.

Under "Proxy Hosts," choose "Add Proxy Host."

Domain Names: glances.host.home

Scheme: http (not "https"...)

Forward Hostname/IP: 192.168.1.235 (IP address of the docker host)

Forward Port: 61208

(I also have "Block Common Exploits" enabled).

Under "SSL," under "SSL Certificate," there should be an option to select the certificate you just imported. Select it.

(I have "Force SSL," "HTTP/2 Support," "HSTS Enabled" enabled.)

On my router, I created a DNS record for host `glances.host.home` with address 192.168.1.235.

Any other services are configured the same way: a proxy host entry in NPM, and a DNS record on the router (pointing to the same IP address of 192.168.1.235). NPM then routes that to the appropriate service.

##### https issues with other devices

I set up [Jellyfin](https://jellyfin.org) as above. But, I couldn't connect to it from the Jellyfin app on my TV. Unchecking "Force SSL" in the NPM proxy host entry for Jellyfin fixed this.

#### more networking

[According to Nginx Proxy Manager Advanced Configuration](https://nginxproxymanager.com/advanced-config/):

> For those who have a few of their upstream services running in docker on the same docker host as NPM, here's a trick to secure things a bit better. By creating a custom docker network, you don't need to publish ports for your upstream services to all of the docker host's interfaces.

Sounds good. For this project, I created `services/zznetwork.yml`:

```yaml
# https://nginxproxymanager.com/advanced-config/#best-practice-use-a-docker-network
networks:
  default:
    external:
      name: jedfmnet
```

* "zz..." so this file gets added last to `docker-compose.yml`, because I'm lazy and didn't want to figure out a better way (the "networks:" section need to be separate from the "services:" section).

A couple of things need to be changed:

1. Any `ports:` settings in the `services/*.yml` files need to be *commented out*. For example, `services/glances.yml`:

    ```yaml
    # ports:
    #   - "61208:61208"
    ```

2. Back in NPM, the proxy host entry changes a bit. The hostname is now the service name ("glances").

> Even though this port isn't listed in the docker-compose file, it's "exposed" by the portainer docker image for you and not available on the docker host outside of this docker network. The service name is used as the hostname, so make sure your service names are unique when using the same network.

#### macvlan network

Pi-hole is annoying in that it needs port 80 and 443, which NPM uses already, but there are at least three ways to get them to work together.

1. Buy another Raspberry Pi to run Pi-hole on.
2. Modify [pi-hole docker](https://github.com/pi-hole/docker-pi-hole) config so it [works with a proxy](https://github.com/pi-hole/docker-pi-hole/blob/master/docker-compose-jwilder-proxy.yml).
3. Put it on a [macvlan network](https://docs.docker.com/network/network-tutorial-macvlan/) so it appears as another device.
4. ?...

##### on the router...

This all depends on how your network is configured. Mine is 192.168.1.0/24, so this is what I did ([ipcalc](http://jodies.de/ipcalc) helps):

The DHCP reservation range on my router is set to:

```
start: 192.168.1.32
  end: 192.168.1.254
```

* 192.168.2 - 192.168.31 available outside of DHCP (192.168.1.1 is the router).

For a range of 192.168.1.16/28:

```
HostMin: 192.168.1.17
HostMax: 192.168.1.30
```

* 192.168.1.2 - 192.168.1.16 are available for static ip assignment
* 192.168.1.17 - 192.168.1.30 are available for docker macvlan.

summary:

```
192.168.1.1 - router
192.168.1.2 - 192.168.1.16 are free for static assignment
192.168.1.17 - 192.168.1.30 - docker macvlan
192.168.1.31 - 192.168.1.254 - router dhcp
192.168.1.255 - broadcast
```

##### on the docker host...

Ok... in `services/zznetwork.yml`, add:

```yaml
  # separate network for pihole, etc.
  # https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/
  wharfnet:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: 192.168.1.0/24
          ip_range: 192.168.1.17/28
          gateway: 192.168.1.1
```

* `eth0` might be called something else, use `ifconfig` to figure it out.

Some additional setup is necessary so the docker host can "see" the macvlan network.

1. Create network "shim" so host can talk to wharfnet:

    ```sh
    sudo ip link add wharfnet-shim link eth0 type macvlan mode bridge
    sudo ip addr add 192.168.1.17/32 dev wharfnet-shim
    sudo ip link set wharfnet-shim up
    sudo ip route add 192.168.1.16/28 dev wharfnet-shim
    ```

    * These *do not* persist across restarts, so add them to `etc/rc.local` or wherever needed to run at system startup.

2. Assign network to the service.

    Add to `services/pihole.yml`:

    ```yaml
    # network must exist
    networks:
      wharfnet:
        ipv4_address: 192.168.1.18
    ```

    `./dock` recreates `docker-compose.yml`.

    `docker-compose up -d pihole` starts Pi-hole.

    Pi-hole appears as 192.168.1.18.

3. On my router, I created a DNS record for `pihole.host.home` with the address `192.168.1.18`.

4. Finally... in NPM... add a proxy host entry for Pi-hole.

    Under "Proxy Hosts," choose "Add Proxy Host."

    Domain Names: pihole.host.home

    Scheme: http (not https...)

    Forward Hostname/IP: 192.168.1.18 (IP address of the macvlan pihole)

    Forward Port: 80

    Add SSL etc., configure as above under [Add Proxy Host](#Add Proxy Host).

## whew

This is way longer than I intended. If you give some time to sink in, you'll figure it out. All I did was search, read, re-read, and experiment until things (somehow) started to work.

Your mileage may vary. I apologize for any unintentional errors.

## more info & sources for this readme

[certificates for localhost](https://letsencrypt.org/docs/certificates-for-localhost/)

[docker docs](https://docs.docker.com/)

[name resolution and dns on home networks](https://stevessmarthomeguide.com/name-resolution-and-dns-on-home-networks/)

[run pihole in docker on ubuntu with reverse proxy](https://www.smarthomebeginner.com/run-pihole-in-docker-on-ubuntu-with-reverse-proxy/)

[reverse proxy with https without opening ports](https://selfhostedhome.com/reverse-proxy-with-https-without-opening-ports/)

[SWAG setup](https://docs.linuxserver.io/general/swag)

[ufw and docker](https://github.com/chaifeng/ufw-docker)

[using docker macvlan networks](https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/)

## inspiration & related

[awesome-docker](https://awesome-docker.netlify.app/)

[DockSTARTer](https://dockstarter.com/)

[docker-veth](https://github.com/samos123/docker-veth)

[linuxserver](https://www.linuxserver.io/)

[LMDS](https://greenfrognest.com/)
