# 1. Install docker-ce
```shell
apt update
apt autoremove --purge
apt full-upgrade -y
apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
apt-key fingerprint 0EBFCD88
add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"

apt update
apt purge --autoremove docker docker-engine docker.io containerd runc
apt install -y docker-ce docker-ce-cli containerd.io
```

---
# 2. [Optional] Change the docker image source to a mirror
**# vim `/etc/docker/daemon.json`**
```json
{
    "dns": ["183.60.83.19"],
    "registry-mirrors": ["https://mirror.ccs.tencentyun.com"]
    "data-root": "/mnt/docker-data",
    "storage-driver": "overlay2"
}
```

Or if you are working on an old system which use SysVinit to manage daemons:  
**# vim `/etc/default/docker`**
```
# Docker Upstart and SysVinit configuration file

#
# THIS FILE DOES NOT APPLY TO SYSTEMD
#
#   Please see the documentation for "systemd drop-ins":
#   https://docs.docker.com/engine/admin/systemd/
#

# Customize location of Docker binary (especially for development testing).
#DOCKERD="/usr/local/bin/dockerd"

# Use DOCKER_OPTS to modify the daemon startup options.
DOCKER_OPTS="--dns 183.60.83.19 --registry-mirror=https://mirror.ccs.tencentyun.com"

# If you need Docker to use an HTTP proxy, it can also be specified here.
#export http_proxy="http://127.0.0.1:3128/"

# This is also a handy place to tweak where Docker's temporary files go.
#export DOCKER_TMPDIR="/mnt/bigdrive/docker-tmp"
```

---
# 3. Get only the necessary files
```shell
mkdir -p /docker/endlessh; mkdir -p /etc/endlessh; mkdir -p /github
cd /github
git clone https://github.com/skeeto/endlessh.git
cd endlessh
mv ./util/smf/endlessh.conf /etc/endlessh/config
mv Dockerfile Makefile endlessh.c /docker/endlessh
rm -rf '/github/endlessh'
```

---
# 4. Make sure the content of `/etc/endlessh/config` is the same as below:
```
# The port on which to listen for new SSH connections.
Port 22

# The endless banner is sent one line at a time. This is the delay
# in milliseconds between individual lines.
Delay 10000

# The length of each line is randomized. This controls the maximum
# length of each line. Shorter lines may keep clients on for longer if
# they give up after a certain number of bytes.
MaxLineLength 32

# Maximum number of connections to accept at a time. Connections beyond
# this are not immediately rejected, but will wait in the queue.
MaxClients 4096

# Set the detail level for the log.
#   0 = Quiet
#   1 = Standard, useful log messages
#   2 = Very noisy debugging information
LogLevel 1

# Set the family of the listening socket
#   0 = Use IPv4 Mapped IPv6 (Both v4 and v6, default)
#   4 = Use IPv4 only
#   6 = Use IPv6 only
BindFamily 0
```

---
# 5. Edit the file `/docker/endlessh/Dockerfile` as below:
```dockerfile
FROM alpine AS builder
RUN apk add --no-cache build-base
# To specify a mirror, use the following commands instead:
# RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tencentyun.com/g' /etc/apk/repositories
# RUN apk add --no-cache build-base

COPY endlessh.c Makefile /
RUN make

FROM alpine AS prod
COPY --from=builder /endlessh /
EXPOSE 22/tcp
ENTRYPOINT ["/endlessh"]
CMD ["-f /etc/endlessh/config >/etc/endlessh/endlessh.log 2>/etc/endlessh/endlessh.err"]
```

---
# 6. Build up docker image and start up a container
```shell
cd /docker/endlessh/
docker build -t i_endlessh .
docker run -d -p 22:22 --name c_endlessh \
--mount type=bind,src=/etc/endlessh,dst=/etc/endlessh \
i_endlessh
docker system prune -af --volumes
```

// In case you are using an alpine package mirror that only intranet accessible, try the following command:  
// As for Tencent CVM users, there are 2 DNS servers can use,  
// One is 183.60.83.19, another one is 183.60.82.98.  
`docker build -t i_endlessh --add-host=mirrors.tencentyun.com:$(dig @183.60.83.19 mirrors.tencentyun.com +short) .`

~~// Somehow the above command doesn't work properly, quite perplexing.  
// In this guide, we solved the problem by specifying the DNS server used for docker containers in Step 2. (Or you can also type command `dockerd --dns 183.60.83.19` if you are not about to edit the daemon configuration file)  
// And there are two other ways to make containers be able to solve hosts which can only be solved in an intranet. Refer to the comment [here](https://github.com/moby/moby/issues/5779#issuecomment-478518282).~~

**// WTF. Just remember one thing: the f\*\*king Tencent Mirrors doesn't provide an HTTPS service. I spent lots of time and tried in vain to check the DNS settings…**

---
# 7. Append a crontab rule to make the container been restarted while the system has rebooted.
**# sudo crontab -e -u root**
```
@reboot docker restart c_endlessh > /dev/null 2>&1 &
```

---
# 8. In some case, the following commands might be helpful:
```shell
docker container logs c_endlessh
docker exec c_endlessh pkill -15 endlessh
```
