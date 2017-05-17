# Requirements

> **Note**: The following instructions have been conformed taking into account that PWD will be deployed in a Linux server.   

## Kernel modules

`overlay` and `xt_ipvs` are necessary in order for PWD to work. You can load the modules directly using `sudo modprobe <module_name>` or add them to /etc/modules so they're loaded automatically once the system boots.

## Install docker
btw, that is not what I wanted to show you ;P. 
Follow the instructions from the [official docker repo](https://docs.docker.com/engine/installation/) to install the CE docker engine for your specific distro. 

## Configure devicemapper storage with `direct-lvm`

Follow the [storage driver documentation](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#configure-direct-lvm-mode-for-production) to configure your daemon to use devicemapper with `direct-lvm`.

Don't forget to remove your graph folder (usually /var/lib/docker) after this step or your daemon will throw an error when you start it afterwards.

## Configure your daemon to use devicemapper and proper DNS forwarded
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

## Remove search DNS entries

Remove any `search` entry in your resolv.conf file.

## Start your docker daemon and set it in swarm mode

```bash
sudo service docker start
docker swarm init
```


# Cleaning up everything PWD creates

```bash
# Deletes all the exited or created containers with it's associated volume
docker rm -fv $(docker ps -aq --filter status=exited --filter status=created)

# Removes all PWD networks
docker network rm $(docker network ls | grep "-" | cut -d ' ' -f1)
```