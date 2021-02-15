# jedfm (just enough docker for me)

A simple docker-compose helper script. All it really does is combine `.yml` files into a `docker-compose` file.

I'm using a Raspberry Pi 4 with latest Raspbian installed.

## setup

Install `docker-ce` per your system.

```sh
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

```
volumes:
  - /etc/localtime:/etc/localtime:ro
```

Otherwise, set `DOCK_TZ=America/New_York` to the appropriate timezone in `./.env` and comment out or remove this entry under `volumes:`

```
# - /etc/localtime:/etc/localtime:ro in services/*
```

* I just noticed that `echo $TZ` doesn't return anything, even though `/etc/environment` contains `TZ=:/etc/localtime`. I'm pretty sure this was working before... `/usr/lib/environment.d/99-environment.conf` is symlinked to `/etc/environment`... at anyrate, `export $TZ=:/etc/localtime` takes care of it for now...

## usage

### summary

```
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

```sh

./dock

```

`./dock -c` also runs `docker-container config` to check the configuration.

### start

```
docker-compose up -d [name]
```

### disable

```
./dock -d [name ...]
```

No arguments shows what's disabled.

### enable

```
./dock -e [name ...]
```

No arguments shows what's enabled.

### generate password

```
./dock -g [length]
```

Just a convenience for services that need new passwords.

### ports

List ports defined in `services/*.yml` (check for duplicate ports in different services):

```
./dock -p
```

### ctop

[ctop](https://github.com/bcicen/ctop) is a nice, simple way to see what's going on.

```
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
alias dcp='docker-compose'
alias dip="docker ps -q | xargs -n 1 docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} {{ .Name }}' | sed 's/ \// /'"

# completion
complete -F _docker d
complete -F _docker_compose dcp
```

[linuxserver.io tips & tricks](https://docs.linuxserver.io/general/docker-compose#tips-and-tricks)

## configuration notes

### host networking

From [docker docs - Use host networking](https://docs.docker.com/network/host/):

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

## inspiration & related

[awesome-docker](https://awesome-docker.netlify.app/)

[DockSTARTer](https://dockstarter.com/)

[docker-veth](https://github.com/samos123/docker-veth)

[linuxserver](https://www.linuxserver.io/)

[LMDS](https://greenfrognest.com/)
