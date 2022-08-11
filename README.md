# wsl-docker-setup

Set-up docker engine within WSL Ubuntu instance:

First of all check if your WSL isntance is version 2 \
If not, update to WSL version 2 first

```PowerShell
 wsl --set-default-version 2
```

If you experience network issues, below might help.
```sh
echo -e "[network]\ngenerateResolvConf = false" | sudo tee -a /etc/wsl.conf
sudo unlink /etc/resolv.conf
echo nameserver 1.1.1.1 | sudo tee /etc/resolv.conf
```

> Just note that nameserver should be your actual DNS IP address instead of `1.1.1.1`

If your WSL instance is not fresh and you dealt with docker earlier, remove it.
```sh
sudo apt remove docker docker-engine docker.io containerd runc
```

Next update packages

```sh
sudo apt update && sudo apt upgrade
```

Install dependencies
```sh
sudo apt install --no-install-recommends apt-transport-https ca-certificates curl gnupg2
```

Switch to legacy iptables, in order to allow docker engine manage FW.\
After running below, select iptables-legacy if it's not selected already.
```sh
update-alternatives --config iptables
```

Set  OS specific environment variables by dot-sourcing os-releases file
```sh
. /etc/os-release
```

Make sure apt will trust docker repo by executing below:
```sh
curl -fsSL https://download.docker.com/linux/${ID}/gpg | sudo tee /etc/apt/trusted.gpg.d/docker.asc
```

Add docker repo
```sh
echo "deb [arch=amd64] https://download.docker.com/linux/${ID} ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
```

Install docker via apt
```sh
sudo apt install docker-ce docker-ce-cli containerd.io
```

Add user to docker group
```sh
sudo usermod -aG docker $USER
```

In order to start docker run below 
```sh
sudo -b dockerd > /dev/null 2>&1
```

----
Below steps are optional but having ability to manage docker from all WSL distros you use, is quite convenient.\
First of all we need to ensure that group ID for docker group will be the same across all WSL distributions.\
Below steps are applied only to machine, which is going to be master running docker service.\
Other distributions will connect to this one via shared socket.

This can be achieved by following:

Check for group ID which is not in use.
In my example, I'll use 36257 but you can use any other which is free

```sh
getent group | grep 36257
```

If above doesn't return you anything, this means group ID is free.
If it returns something along those lines `<group_name>:x:36257:<some users>` find another ID.
However if it's empty you're ready to go :)

Update group ID for docker group which has been created earlier
```sh
sudo sed -i -e 's/^\(docker:x\):[^:]\+/\1:36257/' /etc/group
```

> Note that you need to update group ID for every WSL distribution, you want to run docker commands.

Prepared shared drive which is going to be accessible by all WSL distributions
```sh
DOCKER_DIR=/mnt/wsl/shared-docker
mkdir -pm o=,ug=rwx "$DOCKER_DIR"
chgrp docker "$DOCKER_DIR"
```
Configure docker daemon to use shared directory.
In order to do that you need to create config file under `/etc/docker` directory.

```sh
mkdir -p /etc/docker
touch /etc/docker/daemon.json
```

Add below to `/etc/docker/daemon.json` file. You can use vi/vim/nano/whatever suits you.
```json
{
  "hosts": ["unix:///mnt/wsl/shared-docker/docker.sock"]
}
```

Configure passwordless execution of dockerd and service
Run `sudo visudo` and add following entries at the end of file
```
%docker ALL=(ALL)  NOPASSWD: /usr/bin/dockerd
```

Create shell script which is going to setup and start docker automatically.\
Add below content to file, for instance `~/bin/docker-service`

```sh
DOCKER_DISTRO="Ubuntu"
DOCKER_DIR=/mnt/wsl/shared-docker
DOCKER_SOCK="$DOCKER_DIR/docker.sock"
export DOCKER_HOST="unix://$DOCKER_SOCK"
if [ ! -S "$DOCKER_SOCK" ]; then
    mkdir -pm o=,ug=rwx "$DOCKER_DIR"
    chgrp docker "$DOCKER_DIR"
fi

/mnt/c/Windows/System32/wsl.exe -d $DOCKER_DISTRO sh -c "nohup sudo -b dockerd < /dev/null > $DOCKER_DIR/dockerd.log 2>&1"
```

Change mode for above file so it can be executed. This is done via `chmod`\
For instance: `chmod +x ~/bin/docker-service`

Add to your .bashrc or .profile reference to above script.
```
. ~/bin/docker-service
```

If everything is configured correctly, you can close and open new WSL window.\
Docker should be up and running. 

----

In order to validate if docker is configured properly run below command :

```sh
docker run --rm hello-world
```
This should result in something similar to:

```sh
$ docker run --rm hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete
Digest: sha256:7d246653d0511db2a6b2e0436cfd0e52ac8c066000264b3ce63331ac66dca625
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

----
This part is optional if you chose to use other WSL distribution beside Ubuntu.
In short what you need to do is install docker and set DOCKER_HOST variable to point to shared socket.
Below example is base on `Debian GNU/Linux 9.13 (stretch)` :

- Install docker
```sh
sudo apt update && sudo apt upgrade
sudo apt install --no-install-recommends apt-transport-https ca-certificates curl gnupg2
. /etc/os-release
curl -fsSL https://download.docker.com/linux/${ID}/gpg | sudo tee /etc/apt/trusted.gpg.d/docker.asc
echo "deb [arch=amd64] https://download.docker.com/linux/${ID} ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
sudo sed -i -e 's/^\(docker:x\):[^:]\+/\1:36257/' /etc/group # Note the same ID 36257 for group ID
```

- Set DOCKER_HOST variable which points to shared socket
```sh
export DOCKER_HOST="unix:///mnt/wsl/shared-docker/docker.sock"
```

> In order to persist DOCKER_HOST, place it in .bashrc or .profile