# Docker Security Cheat Sheet  
    
```sh
~$ docker version
Client version: 1.6.0
Client API version: 1.18
Go version (client): go1.4.2
Git commit (client): 4749651
OS/Arch (client): linux/amd64
Server version: 1.6.0
Server API version: 1.18
Go version (server): go1.4.2
Git commit (server): 4749651
OS/Arch (server): linux/amd64
```

##Docker daemon host
Lock down with a firewall, remove SUID/GUID, password policies, stricter SSH configuration, and so on.  

*Ubuntu/Debian:*  
[Stricter settings for your ubuntu server](http://konstruktoid.net/2014/07/07/ubuntu_config-sh-stricter-settings-for-your-ubuntu-server/)  
[CIS Ubuntu 14.04 LTS Server Benchmark v1.0.0](https://benchmarks.cisecurity.org/downloads/show-single/?file=ubuntu1404.100)  
[StricterDefaults](https://help.ubuntu.com/community/StricterDefaults)  

*RedHat/Fedora:*  
[A Guide to securing Red Hat Enterprise Linux 7](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Security_Guide/)   

*General*  
[Deploy and harden a host with Docker Machine](http://konstruktoid.net/2015/02/23/deploy-and-harden-a-host-with-docker-machine/)

##Docker daemon options  
`--icc=false` Use `--link` on run instead.  
`--selinux-enabled`  
`--default-ulimit` Set strict limits as default, it's overwritten by `--ulimit` on run.  
`--tlsverify` Enable TLS, [Protecting the Docker daemon Socket with HTTPS](https://docs.docker.com/articles/https/).

`$ docker -d --tlsverify --tlscacert=ca.pem --tlscert=server-cert.pem --tlskey=server-key.pem
  -H=0.0.0.0:2376 -icc=false --default-ulimit nproc=512:1024 --default-ulimit nfile=50:100`

##Docker run options 
**Capabilities**  
`--cap-drop=all` Drop all capabilities by default.  
`--cap-add net_raw` Allow only needed  

```sh  
~$ docker run --rm --name name -t image
~$ docker exec `docker ps -q` ps a | awk '{print $1}' | grep -v PID
~$ docker exec `docker ps -q` cat /proc/PID/status | grep CapEff | awk '{print $NF}'
~$ docker exec `docker ps -q` capsh --decode=CAPEFF | sed -e 's/.*=//' -e 's/cap_/--cap-add /g' -e 's/,/ /g'
```

For reference:  
`~$ curl https://raw.githubusercontent.com/torvalds/linux/master/include/uapi/linux/capability.h | grep " CAP_" | awk '{print $2, $3}'`  

**Cgroups**  
`--cgroup-parent` Parent cgroup for the container  

**Devices**  
`--device` Mount read-only if required  

**Labels**  
`--security-opt`  Set the SELinux label or AppArmor profile to be applied to the container.   

**Log and logging drivers**  
`-v /dev/log:/dev/log`   
`--log-driver`  Send container logs to other systems such as Syslog.   

**Memory and CPU limits**  
`--cpuset-cpus` CPUs in which to allow execution (0-3, 0,1).    
` -m, --memory` Memory limit.  
`--memory-swap""` Total memory limit.     
`--ulimit` Set the ulimit on the specific container.

**Time**  
`-v /etc/localtime:/etc/localtime:ro`  

**User**  
`-u, --user` Run as a unprivileged user  

##Dockerfile example
```sh
FROM debian:latest [1]

COPY files/example /tmp/example [2]
ADD https://raw.githubusercontent.com/konstruktoid/Docker/master/Security/cleanBits.sh /tmp/cleanBits.sh [3]

RUN \
	apt-get update && \
	apt-get -y upgrade && \ [4]
	apt-get -y clean && \
	apt-get -y autoremove

RUN \
	useradd --system --no-create-home --user-group --shell /bin/false dockeru && \ [5]
	/bin/bash /tmp/cleanBits.sh

ENTRYPOINT ["/bin/bash"]
CMD []
```

1. Do we trust the remote repository? Is there any reason we're not using a homebuilt base image?  
2. COPY local files  
3. ADD remote files
4. Keep the container up-to-date
5. Use a local unprivileged user account

###Docker run example
`~$ export CAP="--cap-drop all --cap-add net_bind_service --cap-add net_raw"`  

If root user is required:   
`~$ docker run --rm -v /etc/localtime:/etc/localtime:ro -v /dev/log:/dev/log $CAP --name <NAME> -t <IMAGE>`  

Unpriv user if possible:  
`~$ docker run --rm -u dockeru -v /etc/localtime:/etc/localtime:ro -v /dev/log:/dev/log $CAP --name <NAME> -t <IMAGE>` 



