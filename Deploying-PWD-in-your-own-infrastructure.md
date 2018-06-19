# Requirements

> **Note**: The following instructions have been conformed taking into account that PWD will be deployed in a Linux server. Also the environment is lifted up without Go on the host system. You only require Docker, Docker-compose and Golang Dep.  

Make sure you have a (local or remote) domain that you can configure and your clients can reach this domain (clients should use the same DNS server for local domains). There should be a `A` record pointing to the server where you will deploy PWD on. For example play.lan. Also, for external services to be accessed, tehere should be a `CNAME` record for *.play.lan pointing to play.lan (so they will return the same IP).

1. Kernel modules

`overlay` and `xt_ipvs` are necessary in order for PWD to work. You can load the modules directly using `sudo modprobe <module_name>` or add them to /etc/modules so they're loaded automatically once the system boots.

2. Install docker

Follow the instructions from the [official docker repo](https://docs.docker.com/engine/installation/) to install the CE docker engine for your specific distro. 

3. Configure devicemapper storage with `direct-lvm`

Follow the [storage driver documentation](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#configure-direct-lvm-mode-for-production) to configure your daemon to use devicemapper with `direct-lvm`.

Don't forget to remove your graph folder (usually /var/lib/docker) after this step or your daemon will throw an error when you start it afterwards.

4. Configure your daemon to use devicemapper and proper DNS forwarded
Add the following JSON under /etc/docker/daemon.json:

```json
{
    "storage-driver": "devicemapper",
    "storage-opts": [
        "dm.thinpooldev=/dev/mapper/docker-thinpool",
        "dm.use_deferred_removal=true",
        "dm.use_deferred_deletion=true"
    ],
    "dns": [
        "<docker_gwbridge_ip>",
        "8.8.8.8",
        "10.0.0.2"
    ]
}
```
**Note:** Make sure to set the `docker_gwbridge` interface IP in the previous file where it's needed. 

5. Remove search DNS entries

Remove any `search` entry in your resolv.conf file.

6. Start your docker daemon and set it in swarm mode

```bash
sudo service docker start
docker swarm init
```
7. Add the following sysctl configurations to prevent scaling issues:

Add the following lines at the end of /etc/sysctl.conf and then run `sudo sysctl -p`

```bash
net.ipv4.neigh.default.gc_thresh3 = 8192
net.ipv4.neigh.default.gc_thresh2 = 8192
net.ipv4.neigh.default.gc_thresh1 = 4096
fs.inotify.max_user_instances = 10000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.ip_local_port_range = 1024 65000
net.netfilter.nf_conntrack_tcp_timeout_established = 600
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 1
```

8. Clone the project to the right directory

The project should be cloned in Go folder structure. This means you should create $GOPATH/src. Default this is $HOME/go/src.

```bash
mkdir -f $HOME/go/src/github.com/play-with-docker
cd $HOME/go/src/github.com/play-with-docker
git clone https://github.com/play-with-docker/play-with-docker.git
cd play-with-docker
```

Install dep to install required packages and fetch all packages. Download `dep` from https://github.com/golang/dep/releases and place the binary in a location that is in your PATH

```bash
curl -L -O https://github.com/golang/dep/releases/download/v0.4.1/dep-linux-amd64 /usr/local/bin/dep
chmod +x /usr/local/bin/dep

#Install the dependencies
dep ensure -v
```

9. Install docker-compose

Follow the instructions on https://docs.docker.com/compose/install/

10. Set the GOPATH correct. This environment variable is used in Docker-compose to mount the code for Play-with-docker. This should be something like

```bash
export GOPATH="${HOME}/go"
```

11. Set flags in `docker-compose.yml`. Put the flags behind the pwd command. These flags are:

### Flags:

**-playground-domain** - `Default domain name to use for this PWD instance (if not set it will be localhost)`  
**-default-dind-image** - `Default dind image to use when creating an instance, default is franela/dind`  

For all flags use the help page of pwd. You can see the help page with this command:

```bash
docker-compose run pwd bash -xc 'cd /go/src/github.com/play-with-docker/play-with-docker; go run api.go -help'
```

12. Pull dind images

Pull the default PWD `dind` images or your own if needed.
```bash
docker pull franela/dind
```

13. Start the PWD containers with Docker compose

This step will lift op three containers.
- HaProxy - This will route the traffic to the richt container
- PWD - This is the 'main' engine as this serves the viewer and you use it to start the instances
- l2 - A routing container. This is used for exposing external services to route it to the right container.

```bash
cd play-with-docker
docker-compose up -d
```

You can check with `docker-compose logs` if there are no issues when starting.

14. Use your environment

Browse with your favorite browser to your domain. You will see the start page of PWD and a Start-button. When you click on it it will start a new session. If not (HTTP 500) then you may have missed `docker swarm init` command. You can run it and then a new session should be up and running. Then you can create a new instance. It this fails you maybe didn't pull the DIND image. Also these errors will appear in the logging (visible by `docker-compose logs -f`).

By default you can start an ssh session to your instance by using port 8022 (use `-p 8022` behind your SSH-command).

# Cleaning up everything PWD creates

```bash
# Stop and remove the containers
docker-compose kill
docker-compose rm -fv

# Removes all PWD networks
docker network rm $(docker network ls | grep "-" | cut -d ' ' -f1)
```