# 1. Install docker-ce
```shell
. /etc/os-release
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | sudo apt-key add -
sudo apt --autoremove -y purge docker docker-engine docker.io containerd runc  docker-ce docker-ce-cli containerd.io

sudo apt-key del '0EBFCD88'
sudo sed -i '/docker/d' /etc/apt/sources.list

sudo apt update
sudo apt --purge autoremove
sudo apt -y full-upgrade
sudo apt -y install podman
```

---
# 2. Get only the necessary files
```shell
mkdir -p /docker/endlessh; mkdir -p /etc/endlessh; mkdir -p /github
cd /github
git clone --depth 1 https://github.com/skeeto/endlessh.git
cd endlessh
mv ./util/smf/endlessh.conf /etc/endlessh/config
mv Dockerfile Makefile endlessh.c /docker/endlessh
rm -rf '/github/endlessh'
```

---
# 3. Make sure the content of `/etc/endlessh/config` is the same as below:
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
# 4. Edit the file `/docker/endlessh/Dockerfile` as below:
```dockerfile
FROM alpine AS builder
# To specify a mirror, use the following commands instead:
# RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tencentyun.com/g' /etc/apk/repositories
# RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.cloud.aliyuncs.com/g' /etc/apk/repositories
RUN apk add --no-cache build-base

COPY endlessh.c Makefile /
RUN make

FROM alpine AS prod
COPY --from=builder /endlessh /
EXPOSE 22/tcp
ENTRYPOINT ["/endlessh"]
CMD ["-f /etc/endlessh/config >/etc/endlessh/endlessh.log 2>/etc/endlessh/endlessh.err"]
```

---
# 5. Build up docker image and start up a container
```shell
cd /docker/endlessh/
podman build -t i_endlessh .
```

// In case you are using an alpine package mirror that only intranet accessible, try the following command:  
// As for Tencent CVM users, there are 2 DNS servers can use,  
// One is 183.60.83.19, another one is 183.60.82.98.  
`podman build -t i_endlessh --add-host=mirrors.tencentyun.com:$(dig @183.60.83.19 mirrors.tencentyun.com +short) .`

// Or if you are one of the Aliyun users (VM must be located in Beijing), there are also 2 DNS servers can use,  
// One is 100.100.2.138, another one is 100.100.2.136.  
`podman build -t i_endlessh --add-host=mirrors.cloud.aliyuncs.com:$(dig @100.100.2.138 mirrors.cloud.aliyuncs.com +short) .`

```shell
podman run --restart=unless-stopped --name c_endlessh \
           --mount type=bind,src=/etc/endlessh,dst=/etc/endlessh \
           -d -p 22:22 \
           i_endlessh

podman generate systemd -f -n c_endlessh --restart-policy=always
sudo mv container-c_endlessh.service /etc/systemd/system
systemctl daemon-reload
systemctl enable --now container-c_endlessh.service
systemctl status container-c_endlessh.service

podman system prune -af --volumes
```

---
# 6. In some case, the following commands might be helpful:
```shell
podman container logs c_endlessh
podman exec c_endlessh pkill -15 endlessh
```
