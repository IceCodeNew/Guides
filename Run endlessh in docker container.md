# Install docker-ce
```
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
# Get only the necessary files
```
mkdir -p /docker/endlessh; mkdir -p /etc/endlessh; mkdir -p /github
cd /github && git clone https://github.com/skeeto/endlessh.git
cd endlessh && mv ./util/smf/endlessh.conf /etc/endlessh/config
mv Dockerfile Makefile endlessh.c /docker/endlessh
rm -rf '/github/endlessh'
```
---
# Make sure the content of `/etc/endlessh/config` is the same as below:
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
# Edit the file `/docker/endlessh/Dockerfile` as below:
```
FROM alpine AS builder
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
# Build up docker image and start up a container
```
cd /docker/endlessh/ && docker build -t i_endlessh . && docker image prune
docker run -d -p 22:22 --name c_endlessh \
--mount type=bind,src=/etc/endlessh,dst=/etc/endlessh \
i_endlessh
```
---
# In some case the following commands might be helpful:
```
docker container logs c_endlessh
docker exec c_endlessh pkill -15 endlessh
```

