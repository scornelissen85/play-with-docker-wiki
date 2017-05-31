# Requirements

> **Note**: The following instructions have been conformed taking into account that PWD will be deployed in a Linux server.   

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
7. Pull dind images

Pull the default PWD `dind` images or your own if needed.
```bash
docker pull franela/dind:overlay2
```

8. Add the following sysctl configurations to prevent scaling issues:

Add the following lines at the end of /etc/sysctl.conf and then run `sudo sysctl -p`

```
net.ipv4.neigh.default.gc_thresh3 = 8192
net.ipv4.neigh.default.gc_thresh2 = 8192
net.ipv4.neigh.default.gc_thresh1 = 4096
fs.inotify.max_user_instances = 10000
net.ipv4.tcp_tw_recycle = 1
net.netfilter.nf_conntrack_tcp_timeout_established = 600
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 1
```

9. Start the PWD container

Use the following command to start the PWD container:

```bash
docker run -d \
        -e DIND_IMAGE=franela/dind:overlay2 \
        -e GOOGLE_RECAPTCHA_DISABLED=true \
        -e MAX_PROCESSES=10000 \
        -e EXPIRY=4h \
        --name pwd \
        -p 80:3000 \
        -p 443:3001 \
        -p 53:53/udp \
        -p 53:53/tcp \
        -v /var/run/docker.sock:/var/run/docker.sock -v sessions:/app/pwd/ \
        --restart always \
        franela/play-with-docker:latest ./play-with-docker --name pwd --cname host1 --save ./pwd/sessions
```

Here's a list of the different options you can specify:


### Env variables:

**DIND_IMAGE** - `Default dind image to use when creating an instance`  
**GOOGLE_RECAPTCHA_SITE_KEY** - `GR site key if you're using recaptcha`  
**GOOGLE_RECAPTCHA_SITE_SECRET** - `GR site key secret if you're using recaptcha`  
**APPARMOR_PROFILE** - `Apparmor profile for the dind container`  
**MAX_PROCESSES** - `Max processes that can be executed in the dind. Prevents fork boms`  
**EXPIRY** - `Default session expiration time. This can be overridden through the API endpoint`  

### Flags:

**-cname** - `Default domain name to use for this PWD instance`  
**-save** - `Session storage file`  
**DIND_IMAGE** - `Default dind image to use when creating an instance`  


# Cleaning up everything PWD creates

```bash
# Deletes all the exited or created containers with it's associated volume
docker rm -fv $(docker ps -aq --filter status=exited --filter status=created)

# Removes all PWD networks
docker network rm $(docker network ls | grep "-" | cut -d ' ' -f1)
```