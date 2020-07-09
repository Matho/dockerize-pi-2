# How to run Rails apps with Docker Swarm, GlusterFS, Traefik, PostgreSQL, Patroni, Etcd, Haproxy, PgBouncer, Portainer in HA on Raspberry Pi 4

Read time: 50 minutes

## Intro
Year ago I have written article `How to run Rails apps with Docker on Raspberry Pi 4` located at
[https://github.com/Matho/dockerize_raspberry_pi](https://github.com/Matho/dockerize_raspberry_pi)  
This is another edition, but his time we will setup Docker Swarm, GlusterFS, Portainer, Traefik, Etcd, Patroni, PgBouncer
to ensure High availability across our mini cluster.

My cluster consist of two raspberry pi model v4. One has 4GB ram and another 8GB ram. Alhought it is recommended to create cluster of size at least 3 nodes, I dont have 3 nodes, only 2.
This configuration will not be truely HA, but if one of the device fail, I will have copy of database and images on another node. With GlusterFS, it should be possible also to backup  
the storage pool to some type of cloud.

This tutorial is not the best tutorial in the world, written by some devops expert. Mainly its notes from my trying to setup mini HA on my two Raspberry Pi. Mostly, its notes for me, to  
remember, how I have installed the whole cluster.

## Ubuntu server installation
I have decided to choose Ubuntu os instead of Raspbian. Mainly because in Raspbian, there is very old image for GlusterFS (v5), at this time exists version 7.6.  
Since Ubuntu [is certified](https://ubuntu.com/blog/ubuntu-20-04-lts-is-certified-for-the-raspberry-pi) on raspberry pi, it
should not be problem to run Ubuntu on raspberry pi. (Also during my testing, is seems to be stable)

Download page is located at [https://ubuntu.com/download/raspberry-pi](https://ubuntu.com/download/raspberry-pi)
You can select which release you want to install on rpi. Lets choose Ubuntu 20.04 LTS 64bit editon (I'm going to install it
on rpi 4).

To be able install Ubuntu Server, you need sd card and [Raspberry Pi Imager](https://ubuntu.com/tutorials/how-to-sdcard-ubuntu-server-raspberry-pi#1-overview)
That is program, which will help you to create bootable copy of Ubuntu Server. Follow the steps on latest link.

When you have prepared sd card, you can start installation of Ubuntu Server. Insert the card to rpi. The steps for installation
are located on [this link](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview) .
You can skip `Install a desktop` step, if you want only server edition installed, like me.

## Change hostname
After default installation, the hostname of rpi is set to `ubuntu`. But what if we have 2 rpi? How we would know, which
node is active, in docker gui? So lets change the hostname of second rpi.

To list current hostname, execute:  
`$ hostnamectl`

To change it to name `ubuntu2`, execute:  
`$ sudo hostnamectl set-hostname ubuntu2`

Verify new changes:  
`$ hostnamectl`

## Static IP
It is recommended to assign static IP to your rpi. Lines begining with `$` symbol means, that you need to
  write it in terminal. Open, or create file  
`$ sudo vim /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`

and insert  
`network: {config: disabled}`

Then open  
`$ sudo vim /etc/netplan/50-cloud-init.yaml`

and insert (replace) with this config - change to your settings
```yaml
network:
  ethernets:
    eth0:
      addresses: [10.0.2.3/24]
      gateway4: 10.0.2.1
      nameservers:
        addresses: [217.119.121.225, 8.8.8.8]
  version: 2
```
Attention! Yaml files are indented by two spaces. Do not use tab, only spaces. If your yaml is not properly
idented, it will raise error.

To submit changes, run  
`$ sudo netplan apply`

You can review the assigned ip via command  
`$ ip add show `

But probably you will be disconnected from ssh session. So login again to your rpi and verify the ip. Also,
you can try to reboot the rpi to verify, the changes are persistent.

## SSH settings
It is recommended to use ssh keys instead of password. With this way, you will not need to insert password each
time you will login to rpi.

Create folder for ssh keys, if do not exists:  
`$ mkdir ~/.ssh`

Verify, if this file is created, if not, create  
`$ vim ~/.ssh/authorized_keys`

Copy paste here your public ssh key (run from your notebook, not rpi:  
`$ cat ~/.ssh/id_rsa.pub`

and insert to rpi  
`$ vim ~/.ssh/authorized_keys`

It is not safe to use default port 22, so we will change it to 7777 (use the value you want). Open  
`$ sudo vim /etc/ssh/sshd_config`

Disable login via password - set this option to no  
`PasswordAuthentication no`

and change the port to 7777, at the begining of file  
`Port 7777`

You can restart your ssh service via  
`$ sudo systemctl restart ssh.service`

It is recommended not to logout from your active shell. Instead, open new terminal window. If you broke the things,
this will ensure, you are able to login to server and you will not end up locked.

Open new terminal and try to login:  
`$ ssh ubuntu@10.0.2.3 -p 7777`

You should be logged in correctly.

## Firewall settings
We will continue with firewall setttings - install small firewall program - `ufw`  
`$ sudo apt-get update`  
`$ sudo apt-get install ufw`

Probably, the ufw is installed already. Lets deny all incoming traffic  
`$ sudo ufw default deny incoming`

Allow required ports
- for ssh: `$ sudo ufw allow 7777`
- for monit: `$ sudo ufw allow 2812`
- for gitlab registry: `$ sudo ufw allow 5005`
- for https: `$ sudo ufw allow 443`
- for http: `$ sudo ufw allow 80`

Enable firewall via cmd  
`$ sudo ufw enable`

The good way is to verify, if the ports are really opened, or blocked. Go to your notebook terminal shell, and install `nmap`  
`$ sudo apt-get install nmap`

Then run command, which will scan all open ports on rpi  
`$ sudo nmap 10.0.2.3`

You should see the similar response:
```txt
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-04 17:20 CEST
Nmap scan report for 10.0.2.3  
Host is up (0.00048s latency).  
Not shown: 997 filtered ports  
PORT     STATE  SERVICE  
80/tcp   closed http  
443/tcp  closed https  
7777/tcp open   cbt  
```

## Swap configuration
Follow the good tutorial at [https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04) .
Let's use 8GB swap file for rpi with 8GB and 4GB for rpi with 4GB ram.

## GlusterFS
This steps are based on tutorial located at [https://www.digitalocean.com/community/tutorials/how-to-create-a-redundant-storage-pool-using-glusterfs-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-create-a-redundant-storage-pool-using-glusterfs-on-ubuntu-20-04)
The main differences are, that we do not have 3 nodes, but only 2 nodes. So, I decided we will have 2 servers and 2 clients.

```text
gluster1	Server, client
gluster2	Server, client
```
Because one of my rpi is running in production, I have only 1 available rpi and the second node is my amd64 arch old notebook. So I tried to do all the steps on amd64 host first, and after the whole tutorial,
I went throught it second time, to install it on the second rpi.

On all nodes, run  
`$ sudo vim /etc/hosts`

and insert this config:
```text
10.0.2.3 gluster1.example.com gluster1
10.0.2.4 gluster2.example.com gluster2
```
Then, insert ppa to install latest 7.6 version on all nodes:  
`$ sudo add-apt-repository ppa:gluster/glusterfs-7`  
`$ sudo apt update`

Then we are ready to install glusterfs server. Because we have 2 gluster servers, install it on both nodes:  
`$ sudo apt install glusterfs-server`

Then also on both nodes start, activate on start and show status of service via:  
`$ sudo systemctl start glusterd.service`  
`$ sudo systemctl enable glusterd.service`  
`$ sudo systemctl status glusterd.service`

Then we want to set firewall. On gluster1 node, do  
`$ sudo ufw allow from 10.0.2.4 to any port 24007`

On gluster2 node, do  
`$ sudo ufw allow from 10.0.2.3 to any port 24007`

On both nodes, deny connection from public:  
`$ sudo ufw deny 24007`

On gluster1:  
`$ sudo gluster peer probe gluster2`

On both nodes:  
`$ sudo gluster peer status`

On gluster1:  
`$ sudo gluster volume create volume1 replica 2 gluster1.example.com:/gluster-storage gluster2.example.com:/gluster-storage force`  
`$ sudo gluster volume start volume1`

On gluster1:  
`$ sudo ufw allow from 10.0.2.4 to any port 49152`

On gluster2:  
`$ sudo ufw allow from 10.0.2.3 to any port 49152`

On both nodes:  
`$ sudo ufw deny 49152`

Because we have 2 clients, run on both nodes:  
`$ sudo apt install glusterfs-client`

On gluster1:  
`$ sudo mkdir /storage-pool`

On gluster2: (only if you try on amd64)  
`$ sudo ln -s /media/martin/sdcard/storage-pool /storage-pool`

On gluster1:  
`$ sudo mount -t glusterfs gluster1.example.com:/volume1 /storage-pool`

On gluster2:  
`$ sudo mount -t glusterfs gluster2.example.com:/volume1 /storage-pool`

On gluster1:  
`$ sudo touch file_{0..9}.test`

On both nodes  
`$ ls /gluster-storage`

On gluster2 (only for amd64 notebook, where I'm running sd card for storage):  
`$ sudo mv /gluster-storage /media/martin/sdcard/gluster-storage`  
`$ sudo ln -s /media/martin/sdcard/gluster-storage /gluster-storage`


`$ sudo systemctl start glusterd.service`

On gluster1:  
`$ sudo gluster volume set volume1 auth.allow 10.0.2.4`

On gluster2:  
`$ sudo gluster volume set volume1 auth.allow 10.0.2.3`

On both nodes:  
`$ sudo gluster volume info`  
`$ sudo gluster peer status`

On both:  
`$ sudo gluster volume profile volume1 start`  
`$ sudo gluster volume profile volume1 info`  
`$ sudo gluster volume status`

Because I first setupped the Gluster on amd64 notebook, I need to found way how to reactivate GlusterFs
on the second rpi. I did a lot of hacking and not sure, if all this commands were correct, but in the end
I solved it.

Run this only, if you setuped GlusterFS on amd64 first, and this is case for your second rpi. If you setup both rpi during previous steps, you can skip this section.

At the begining, I need to removed the second brick from the Gluster  
`$ sudo gluster volume remove-brick volume1 replica 1 gluster2.example.com:/gluster-storage force`

Then, it should be possible to detach old gluster2 node, executing from node1:  
`$ sudo gluster peer detach gluster2`

Not sure if needed, but I did restart on both nodes:  
`$ sudo systemctl restart glusterd.service`  
`$ sudo systemctl status glusterd.service`

On first node:  
`$ sudo gluster peer probe gluster2`

On second node:  
`$ sudo gluster peer probe gluster1`

If you do not have success responses, something is bad.  
Show output with some info about volumes:  
`$ sudo gluster volume info volume1`  
`$ sudo gluster peer status`

Then from node1  
`$ sudo gluster volume add-brick volume1 replica 2 gluster2.example.com:/gluster-storage force`

Try to write some files to `/storage-pool` and test, if the file sharing works correctly.

## Docker engine
No we will install Docker. Prerequisites you can find at [https://docs.docker.com/engine/install/ubuntu/#prerequisites](https://docs.docker.com/engine/install/ubuntu/#prerequisites)

Relod system:  
`$ sudo apt-get update`

Install docker dependencies:
```bash
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

Add ppa (change arch if you are install to amd64):  
`$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
```bash
$ sudo add-apt-repository \
   "deb [arch=arm64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

Note the arch=arm64 - it is ppa for Raspberry Pi.

Reload the system again, to fetch new ppa:  
`$ sudo apt-get update`

And install docker:  
`$ sudo apt-get install docker-ce docker-ce-cli containerd.io`

Test installation via running hello-world image:  
`$ sudo docker run hello-world`

## Docker Compose
The Docker Compose overview page is located at [https://docs.docker.com/compose/](https://docs.docker.com/compose/)

Installation page is located at [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

The easiest method is to install via apt-get  
`$ sudo apt-get install docker-compose`

Echo the version to detect, if it is installed correctly:  
`$ docker-compose --version`


## Docker swarm
The Docker swarm tutorial you can find at [https://docs.docker.com/engine/swarm/swarm-tutorial/](https://docs.docker.com/engine/swarm/swarm-tutorial/)

One machine, will be manager, and the second will be worker. The manager node can automatically be worker -
you do not need to join manager node to be worker, it is by default.

The following ports must be available:
- TCP port 2377 for cluster management communications
  - `$ sudo ufw allow 2377`
- TCP and UDP port 7946 for communication among nodes
  - `$ sudo ufw allow 7946`
- UDP port 4789 for overlay network traffic
  - `$ sudo ufw allow 4789`

Lets create manager node:  
`$ sudo docker swarm init --advertise-addr 10.0.2.3`

You will see similar response to this one:
```text
Swarm initialized: current node (ft3x4xybgoodf5wzsn62tn48y) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2xg6qiiqmvbksssv4a4orohjhs5lnrfn1tz4r4y2j9vtk-aplduj12gjdyleocrtiyac5bx 10.0.2.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

If you want to add worker node to swarm, execute:  
`$ sudo docker swarm join --token SWMTKN-1-sdasdasqwqw564564-aplduj12gjdyleocrtiyac5bx 10.0.2.3:2377`

Note: the token will be different.

If you are installing to second rpi, after you have installed docker to amd64, remove the amd64 device from docker swarm first, via:  
`$ sudo docker node rm --force tsilmz37t9t4y7xfua4g3oco8`  
The node id you will get upon running  
`sudo docker node ls`

To detect join worker token, run on manager node:  
`$ sudo docker swarm join-token worker`

Then join from new worker:  
`$ sudo docker swarm join --token SWMTKN-1-4kujz7hw7cn3h9adsadqw465qnic9xot3wy00fbw792iw17-aoczo3jrpwn6v180tfbkbk6wv 10.0.2.3:2377`

Check, if worker was added, on manager node:  
`$ sudo docker node ls`

## Portainer

Portainer is swarm GUI. It make easier to manage your whole swarm cluster. It also has swarm node visualization screen.

To be able run portainre, allow this ports:  
`$ sudo ufw allow 9000`  
`$ sudo ufw allow 8000`

Lets install it on swarm manager node.

Create folder for portainer data. Note: we are referencing to storage-pool. It means, the portainer data  
will be shared accross our GlusterFS, so will be available on both nodes.

`$ sudo mkdir /storage-pool/portainer_data`

Lets start Portainer, via:
```bash
$ sudo docker service create \  
    --name portainer \    
    --publish 9000:9000 \  
    --publish 8000:8000 \  
    --replicas=1 \  
    --constraint 'node.role == manager' \  
    --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \  
    --mount type=bind,src=//storage-pool/portainer_data,dst=/data \  
    portainer/portainer
```

For now, it is deployed without ssl. But configure with ssl is recommended.
Now on your notebook, in the same network, open in your browser url `http://10.0.2.3:9000/`

At the beginning, you will be asked to enter your password. After it, you can browser your docker swarm stack.
The docker visualizer can be found at `http://10.0.2.3:9000/#/swarm/visualizer`

The deploy Portainer readme is located at [https://portainer.readthedocs.io/en/latest/deployment.html](https://portainer.readthedocs.io/en/latest/deployment.html)

## Postgres HA - Spilo, Patroni, Etcd, Pgadmin4
To have PostgreSQL high availability (db replication), the good way is to use Patroni and etcd. To automatize the installation,  
there is pre builded image called `Spilo`. It contains Patroni with etcd.

Spilo can be found at [https://github.com/zalando/spilo/releases](https://github.com/zalando/spilo/releases)
At the time of writing, the last version is 1.6-p3. The problem is, that the Dockerfile in Spilo is prepared
only for amd64 architecture. Because we are running arm64, we need to slightly modify the Dockerfile.

Download the Spilo to your raspberry pi. Prepare build for arm64  
`$  sudo docker build --build-arg COMPRESS=true --tag 'v1.6-p3_aarm64' -t registry.gitlab.matho.sk:5005/root/spilo .`

Soon you will see, that it will fail on 400th line with downloading x86 arch of `wal-g`. Therefore, I have forked te wal-g repo
located at [https://github.com/Matho/wal-g](https://github.com/Matho/wal-g) and rebuild the wal-g binary for arm64. The binary
is pushed to release page at [https://github.com/Matho/wal-g/releases/tag/v0.2.16](https://github.com/Matho/wal-g/releases/tag/v0.2.16)

If you do not trust me, you can build your own version, via the following steps.

I checkouted to git version `v0.2.16` and tryied to build binary for arm64 architecture. I followed the steps located
at [https://github.com/wal-g/wal-g/blob/master/PostgreSQL.md](https://github.com/wal-g/wal-g/blob/master/PostgreSQL.md)
Probably, I have build only support for PostgreSQL

At first, install this required native dependencies:  
`$ sudo apt-get install build-essential make golang golint libsodium-dev cmake liblzo2-dev`

Then expose GOBIN env variable via:  
`$ export GOBIN=~/go/bin`

Go to downloaded wal-g directory:  
`$ cd wal-g`

And build the project via commands:  
`$ make install`  
`$ make deps`  
`$ make pg_install`

The binary output will be located at `~/go/bin/wal-g`. Upload it somewhere accessible and modify the line in Spilo Dockerfile to
download the binary. Note: the binary on release page is not zipped, so do not try to extract it.

Modify the 400th line, replace with:  
`curl -L https://github.com/Matho/wal-g/releases/download/v0.2.16/wal-g_aarm64_v0.2.16 --output /usr/local/bin/wal-g`

Now, there will be another x86 hardcoded settings. Find line:  
`&& libs="$(ldd $files | awk '{print $3;}' | grep '^/' | sort -u) /lib/x86_64-linux-gnu/ld-linux-x86-64.so.* /lib/x86_64-linux-gnu/libnsl.so.* /lib/x86_64-linux-gnu/libnss_compat.so.*" \`

and replace it with:  
`&& libs="$(ldd $files | awk '{print $3;}' | grep '^/' | sort -u) /lib/aarch64-linux-gnu/ld-linux-aarch64.so.* /lib/aarch64-linux-gnu/libnsl.so.* /lib/aarch64-linux-gnu/libnss_compat.so.*" \`

Also, near the end of file, find this lines:
```shell
&& /usr/bin/busybox sh -c "(find $save_dirs -not -type d && cat /exclude /exclude && echo exclude) | sort | uniq -u | xargs /bin/busybox rm" \
&& /usr/bin/busybox--install -s \
&& /usr/bin/busybox sh -c "find $save_dirs -type d -depth -exec rmdir -p {} \; 2> /dev/null"; \
```
and comment it. I had some issues with busybox, so I fixed it by commenting that lines. It did some cleaning, so due to it  
 the folders are not cleaned properly and the docker image is too big.

There is another place with x86 binary. Check for the line 24th:
```
curl -L https://github.com/coreos/etcd/releases/download/v${ETCDVERSION}/etcd-v${ETCDVERSION}-linux-amd64.tar.gz \
| tar xz -C /bin --strip=1 --wildcards --no-anchored etcdctl etcd; \
```
I have build etcd from source, if you trust me, you can replace that line with:
```
curl -L https://github.com/Matho/etcd/releases/download/v2.3.8/etcd_v2.3.8_aarm64.zip -o etcd.zip  \
&& unzip etcd.zip -d /bin \
&& curl -L https://github.com/Matho/etcd/releases/download/v2.3.8/etcdctl_v2.3.8_aarm64.zip -o etcdctl.zip  \
&& unzip etcdctl.zip -d /bin; \
```
Before running build, add `unzip` programm to apt-get at the beginning of Dockerfile.

I you want to build etcd from source, follow this steps:  
`$ git clone https://github.com/coreos/etcd.git`  
`$ cd etcd`  
`$ git checkout v2.3.8`

Open build file:  
`$ vim build`

and change the GIT_SHA line to:
```
`GIT_SHA=`echo 7e4fc7eaa931298732c880a703bccc9c177ae1de || echo "GitNotFound"`
```
It means, that instead of build from master, it will build from v2.3.8 tagged version
Run build:  
`$ ./build`

Builded binaries are under bin folder, in this etcd project.

Note: according to docs, the arm64 arch is in [experimental status](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/supported-platform.md#32-bit-and-other-unsupported-systems) .

To be able run etcd on arm64, you need to prefix call with env variable:
`$ ETCD_UNSUPPORTED_ARCH=arm64 bin/etcd --version`

Here is some opened issue  - [https://github.com/etcd-io/etcd/issues/10677](https://github.com/etcd-io/etcd/issues/10677)

Try to rerun arm64 docker Spline build via:
`$ sudo docker build --build-arg COMPRESS=true --tag 'v1.6-p3_aarm64' -t registry.gitlab.matho.sk:5005/root/spilo .`

When the image is builded, push it to our Gitlab:  
`$ sudo docker login registry.gitlab.matho.sk:5005 -u root -p token`  
`$ sudo docker tag registry.gitlab.matho.sk:5005/root/spilo registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_aarm64`  
`$ sudo docker push registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_aarm64`

Now we can prepare Spilo docker swarm config:
This config is for case, when you run rpi + amd64 device.
```yaml
version: "3.6"

networks:
  db-nw:
    driver: overlay
    
services:
  my_spilo1:
    image: registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_aarm64
    networks:
      db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "5432:5432"
      - "8008:8008"
    volumes:
      - "/data/my_spilo/etcd1:/etc/service/etcd"
      - "/data/my_spilo/spilo1:/home/postgres/"
      - "/data/my_spilo/etcd1:/tmp"        
    environment:
      - PATRONI_POSTGRESQL_LISTEN=0.0.0.0:5432
      - ETCD_UNSUPPORTED_ARCH=arm64
      - SCOPE=sftsscope       
    deploy:
      replicas: 1

  pgadmin4_1:
    image: biarms/pgadmin4:4.21
    networks:
      db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "5050:5050"        
    volumes:
      - "/data/pgadmin4_1:/pgadmin"   
    environment:
      - PGADMIN_DEFAULT_EMAIL=user@domain.com 
      - PGADMIN_DEFAULT_PASSWORD=adsadas 
      - PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION=True 
      - PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only!" 
      - PGADMIN_CONFIG_CONSOLE_LOG_LEVEL=10 
    depends_on:
      - my_spilo1
    deploy:
      labels:
        - traefik.http.routers.pgadmin4_1.rule=Host(`pgadmin4_1.docker.matho.sk`)
        - traefik.http.services.pgadmin4_1-service.loadbalancer.server.port=5050 
        - traefik.docker.network=spilo_db-nw  
      replicas: 1    
      
  my_spilo2:
    image: registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_amd64
    networks:
      db-nw:
        ipv4_address: 10.0.2.4
    ports:
      - "5433:5432"
      - "8009:8008"       
    volumes:
      - "/data/my_spilo/etcd2:/etc/service/etcd"
      - "/data/my_spilo/spilo2:/home/postgres/"
      - "/data/my_spilo/etcd2:/tmp"
    environment:
      - PATRONI_POSTGRESQL_LISTEN=0.0.0.0:5432
      - SCOPE=sftsscope
    deploy:     
      replicas: 1
      
  pgadmin4_2:
    image: dpage/pgadmin4:4.23
    networks:
      db-nw:
        ipv4_address: 10.0.2.4
    ports:
      - "5051:5050"               
    volumes:
      - "/data/pgadmin4_2:/var/lib/pgadmin"        
    environment:
      - PGADMIN_DEFAULT_EMAIL=user@domain.com 
      - PGADMIN_DEFAULT_PASSWORD=asdas 
      - PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION=True 
      - PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only!" 
      - PGADMIN_CONFIG_CONSOLE_LOG_LEVEL=10 
    depends_on:
      - my_spilo2       
    deploy:
      labels:
        - traefik.http.routers.pgadmin4_2.rule=Host(`pgadmin4_2.docker.matho.sk`) 
        - traefik.http.services.pgadmin4_2-service.loadbalancer.server.port=5050    
        - traefik.docker.network=spilo_db-nw   
      replicas: 1  
      
  haproxy_1:
    image: haproxy:2.1.7-alpine
    networks:
      db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "8432:7432"
      - "8433:7433"       
    volumes:
      - "/data/haproxy_1:/usr/local/etc/haproxy"   
    depends_on:
      - pgbouncer_1 
    deploy:   
      replicas: 1 

  pgbouncer_1:
    image: zebardy/pgbouncer:latest
    networks:
      db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "7432:5432"     
    volumes:
      - "/data/pgbouncer_1:/etc/pgbouncer" 
    environment:
      TZ: Europe/Prague
      ADMIN_USERS: admin
      DB_HOST: 10.0.2.3
      DB_USER: admin
      DB_PASSWORD: asdsad
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 300
    depends_on:
      - my_spilo1       
    deploy:     
      replicas: 1        
      
  traefik:
    image: traefik:v2.2.1
    command:
      - "--api.dashboard=true"
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=true"
      - "--providers.docker.network=db-nw"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web-secure.address=:443"
      - --certificatesresolvers.le.acme.email=das@sdas.sd
      - --certificatesresolvers.le.acme.storage=/acme.json
      - --certificatesresolvers.le.acme.tlschallenge=true
    ports:
      # The HTTP port
      - 80:80
      # The Web UI (enabled by --api.insecure=true)
      - 8080:8080
      - 443:443
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - db-nw
    deploy:
      labels:
        - "traefik.http.routers.api.rule=Host(`traefik.docker.matho.sk`)"
        - "traefik.http.routers.api.service=api@internal"
        - "traefik.http.routers.api.middlewares=auth"
        - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$hadasbs$$RQuxjJckwdEefErFO4d3m/"
        - "traefik.http.services.dummy-svc.loadbalancer.server.port=9999" 
      placement:
        constraints:
          - node.role == manager           
                  
  haproxy_2:
    image: haproxy:2.1.7-alpine
    networks:
      db-nw:
        ipv4_address: 10.0.2.4
    ports:
      - "8434:7434"
      - "8435:7435"    
    volumes:
      - "/data/haproxy_2:/usr/local/etc/haproxy"         
    depends_on:
      - pgbouncer_2 
    deploy:   
      replicas: 1 
      
  pgbouncer_2:
    image: edoburu/pgbouncer:1.8.1 
    networks:
      db-nw:
        ipv4_address: 10.0.2.4
    ports:
      - "7435:5432"       
    volumes:
      - "/data/pgbouncer_2:/etc/pgbouncer"        
    environment:
      TZ: Europe/Prague
      ADMIN_USERS: admin
      DB_HOST: 10.0.2.4
      DB_USER: admin
      DB_PASSWORD: asdas
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 300 
    depends_on:
      - my_spilo2        
    deploy:     
      replicas: 1
```

This config is for case, when you run aarm64 + aarm64 device
```yaml
version: "3.6"

networks:
  db-nw:
    driver: overlay
    
services:
  my_spilo1:
    image: registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_aarm64
    networks:
      db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "5432:5432"
      - "8008:8008"
    volumes:
      - "/data/my_spilo/etcd1:/etc/service/etcd"
      - "/data/my_spilo/spilo1:/home/postgres/"
      - "/data/my_spilo/etcd1:/tmp"        
    environment:
      - PATRONI_POSTGRESQL_LISTEN=0.0.0.0:5432
      - ETCD_UNSUPPORTED_ARCH=arm64
      - SCOPE=sftsscope       
    deploy:
      replicas: 1

  pgadmin4_1:
    image: biarms/pgadmin4:4.21
    networks:
      db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "5050:5050"        
    volumes:
      - "/data/pgadmin4_1:/pgadmin"   
    environment:
      - PGADMIN_DEFAULT_EMAIL=user@domain.com 
      - PGADMIN_DEFAULT_PASSWORD=asdas 
      - PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION=True 
      - PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only!" 
      - PGADMIN_CONFIG_CONSOLE_LOG_LEVEL=10 
    depends_on:
      - my_spilo1
    deploy:
      labels:
        - traefik.http.routers.pgadmin4_1.rule=Host(`pgadmin4_1.docker.matho.sk`)
        - traefik.http.services.pgadmin4_1-service.loadbalancer.server.port=5050 
        - traefik.docker.network=spilo_db-nw  
      replicas: 1    
      
  my_spilo2:
    image: registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_aarm64
    networks:
      db-nw:
        ipv4_address: 10.0.2.4
    ports:
      - "5433:5432"
      - "8009:8008"       
    volumes:
      - "/data/my_spilo/etcd2:/etc/service/etcd"
      - "/data/my_spilo/spilo2:/home/postgres/"
      - "/data/my_spilo/etcd2:/tmp"
    environment:
      - PATRONI_POSTGRESQL_LISTEN=0.0.0.0:5432
      - SCOPE=sftsscope
    deploy:     
      replicas: 1
      
  pgadmin4_2:
    image: biarms/pgadmin4:4.21
    networks:
      db-nw:
        ipv4_address: 10.0.2.4
    ports:
      - "5051:5050"               
    volumes:
      - "/data/pgadmin4_2:/var/lib/pgadmin"        
    environment:
      - PGADMIN_DEFAULT_EMAIL=user@domain.com 
      - PGADMIN_DEFAULT_PASSWORD=asdas 
      - PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION=True 
      - PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only!" 
      - PGADMIN_CONFIG_CONSOLE_LOG_LEVEL=10 
    depends_on:
      - my_spilo2       
    deploy:
      labels:
        - traefik.http.routers.pgadmin4_2.rule=Host(`pgadmin4_2.docker.matho.sk`) 
        - traefik.http.services.pgadmin4_2-service.loadbalancer.server.port=5050    
        - traefik.docker.network=spilo_db-nw   
      replicas: 1  
      
  haproxy_1:
    image: haproxy:2.1.7-alpine
    networks:
      db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "8432:7432"
      - "8433:7433"       
    volumes:
      - "/data/haproxy_1:/usr/local/etc/haproxy"   
    depends_on:
      - pgbouncer_1 
    deploy:   
      replicas: 1 

  pgbouncer_1:
    image: zebardy/pgbouncer:latest
    networks:
      db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "7432:5432"     
    volumes:
      - "/data/pgbouncer_1:/etc/pgbouncer" 
    environment:
      TZ: Europe/Prague
      ADMIN_USERS: admin
      DB_HOST: 10.0.2.3
      DB_USER: admin
      DB_PASSWORD: adasda
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 300
    depends_on:
      - my_spilo1       
    deploy:     
      replicas: 1        
      
  traefik:
    image: traefik:v2.2.1
    command:
      - "--api.dashboard=true"
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=true"
      - "--providers.docker.network=db-nw"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web-secure.address=:443"
      - --certificatesresolvers.le.acme.email=madas@adas.ss
      - --certificatesresolvers.le.acme.storage=/acme.json
      - --certificatesresolvers.le.acme.tlschallenge=true
    ports:
      # The HTTP port
      - 80:80
      # The Web UI (enabled by --api.insecure=true)
      - 8080:8080
      - 443:443
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - db-nw
    deploy:
      labels:
        - "traefik.http.routers.api.rule=Host(`traefik.docker.matho.sk`)"
        - "traefik.http.routers.api.service=api@internal"
        - "traefik.http.routers.api.middlewares=auth"
        - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$hlasdsabs$$RQuxjJckwdEefErFO4d3m/"
        - "traefik.http.services.dummy-svc.loadbalancer.server.port=9999" 
      placement:
        constraints:
          - node.role == manager           
                  
  haproxy_2:
    image: haproxy:2.1.7-alpine
    networks:
      db-nw:
        ipv4_address: 10.0.2.4
    ports:
      - "8434:7434"
      - "8435:7435"    
    volumes:
      - "/data/haproxy_2:/usr/local/etc/haproxy"         
    depends_on:
      - pgbouncer_2 
    deploy:   
      replicas: 1 
      
  pgbouncer_2:
    image: zebardy/pgbouncer:latest
    networks:
      db-nw:
        ipv4_address: 10.0.2.4
    ports:
      - "7435:5432"       
    volumes:
      - "/data/pgbouncer_2:/etc/pgbouncer"        
    environment:
      TZ: Europe/Prague
      ADMIN_USERS: admin
      DB_HOST: 10.0.2.4
      DB_USER: admin
      DB_PASSWORD: asdas
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 300 
    depends_on:
      - my_spilo2        
    deploy:     
      replicas: 1
```

On node2, login to Gitlab:  
`$ sudo docker login registry.gitlab.matho.sk:5005 -u root -p token`

If you are installing second rpi, do:
```bash
$ sudo docker pull registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_aarm64
$ sudo docker pull biarms/pgadmin4:4.21
$ sudo docker pull haproxy:2.1.7-alpine
$ sudo docker pull zebardy/pgbouncer:latest

$ sudo docker pull registry.gitlab.matho.sk:5005/mathove-projekty/gymplaci.sk:latest
$ sudo docker pull registry.gitlab.matho.sk:5005/mathove-projekty/matho:latest
$ sudo docker pull registry.gitlab.matho.sk:5005/mathove-projekty/ecavkalnica:latest
$ sudo docker pull registry.gitlab.matho.sk:5005/root/msdc:latest
```

If your second device is amd64 based arch, do:  
`$ sudo docker pull registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_amd64`

On both nodes:  
`$ sudo ufw allow 8008`

On node2:  
`$ sudo ufw allow 5051`

On node1:  
`$ sudo mkdir -p /data/my_spilo/etcd1`  
`$ #sudo mkdir -p /data/my_spilo/run1`  
`$ sudo mkdir -p /data/my_spilo/spilo1/pgdata`  
`$ sudo mkdir -p /data/pgadmin4_1`

On node2:  
`$ sudo mkdir -p /data/pgadmin4_2`  
`$ sudo mkdir -p /data/my_spilo/etcd2`  
`$ sudo mkdir -p /data/my_spilo/spilo2/pgdata`  
`$ sudo mkdir -p /data/my_spilo/etcd2`

On manager:  
`$ sudo docker stack deploy --compose-file spilo-compose.yml spilo`

If there are some issues, run:  
`$ sudo docker swarm ca --rotate`

Run the spilo yaml stack:  
`$ sudo docker stack deploy --compose-file spilo-compose.yml spilo`

To be able start etcd on node1 (and arm64 node2), login to spilo1 container on node1 and find
`$ vim.tiny /etc/runit/runsvdir/default/etcd/run`

To run the etcd on arm64, you need to prefix env variable before running the cmd. Do this on both arm64 devices. Replace the file with:
```bash
#!/bin/sh -e

exec 2>&1
exec env -i ETCD_UNSUPPORTED_ARCH=arm64 /bin/etcd --data-dir /run/etcd.data
```

Do not create symlink to that file. Instead, create copy of that file in:
`/etc/service/etcd/run`

Add execute bit:  
`$ chmod +x /etc/service/etcd/run`

Note: Because I temporary use on second node my old notebook with arch amd64, I'm using different docker image for node2 and
  for node1, also restricted with IP addresses.  Also note, that the volumes are set to /data folder, so it means, that
   postgres data folder is not synchronized via Gluster. It is because of speed.



On node2, I have following error on startup in logs:
```text
ERROR  : Failed to create the directory /var/lib/pgadmin/sessions:
[Errno 13] Permission denied: '/var/lib/pgadmin/sessions
```
The problem was, that on docker data folder on node2, I have symlink to my sdcard. And that card is mounted with nosuid. Write:
`$ mount`

The response is:  
`dev/mmcblk0p1 on /media/martin/sdcard type ext4 (rw,nosuid,nodev,relatime,uhelper=udisks2)`

You can safely remount the drive via:  
`$ sudo mount -n -o remount,suid /media/martin/sdcard`

I recommend to restart docker on node2:  
`$ sudo systemctl restart docker.service`

To detect credentials for PostgreSQL, login to container:  
`$ sudo docker container exec -it <spilo container> bash`

and in working postgres directory, find postgres.yml:  
`$ cat ~/postgres.yml`

You will see this default setttings:
```text
  password: standby
  username: standby
superuser:
  password: zalando
  username: postgres
```

On node1, execute:  
`$ sudo chown -R 5050:5050 /data/pgadmin4_1/`

On node2, execute:  
`$ sudo chown -R 5050:5050 /data/pgadmin4_2/`

On both nodes:  
`$ sudo ufw allow 6789`

The pgadmin will be available on `10.0.2.3:5050` and `10.0.2.4:5051`
You can see, that we use different images for rpi and amd64 device.
Login to pgadmin on both nodes. And create configs. After you restart docker stack, your pgadmin configs should be persisted.
Try to create new database on first node, it should be copied to second node.

# Haproxy, PgBouncer

On node1:  
`$ sudo mkdir /data/pgbouncer_1`

Add Pgbouncer config:  
`$ sudo vim /data/pgbouncer_1/pgbouncer.ini`

```text
[databases]
* = host=10.0.2.3 port=5432 user=admin
[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 5432
unix_socket_dir =
user = postgres
auth_file = /etc/pgbouncer/userlist.txt
auth_type = md5
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 300
ignore_startup_parameters = extra_float_digits

# Log settings
admin_users = admin
# Connection sanity checks, timeouts
# TLS settings
# Dangerous timeouts
```

On node2:  
`$ sudo chmod 777 /data/pgbouncer_2/`

Insert the same pgbouncer config on second node - change only host IP to 10.0.2.4

On node1:
`$ sudo mkdir /data/haproxy_1`

On node2:
`$ sudo mkdir /data/haproxy_2`

To install haproxy, do first on node1s:
`$ cd /data/haproxy_1`  
`$ sudo vim haproxy.cfg`

Insert this config:
```text
listen postgres_write
    bind *:7432
    mode            tcp
    option httpchk
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server postgresql_1 10.0.2.3:5432 check port 8008
    server postgresql_2 10.0.2.4:5432 check port 8008

listen postgres_read
    bind *:7433
    mode            tcp
    balance leastconn
    option pgsql-check user admin
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server postgresql_1 10.0.2.3:5432
    server postgresql_2 10.0.2.4:5432
```

On node2:  
`$ cd /data/haproxy_2`  
`$ sudo vim haproxy.cfg`

```text
listen postgres_write
    bind *:7434
    mode            tcp
    option httpchk
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server postgresql_1 10.0.2.3:5432 check port 8008
    server postgresql_2 10.0.2.4:5432 check port 8008

listen postgres_read
    bind *:7435
    mode            tcp
    balance leastconn
    option pgsql-check user admin
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server postgresql_1 10.0.2.3:5432
    server postgresql_2 10.0.2.4:5432
```
On node1:  
`$ sudo ufw allow 8432`

On node2:  
`$ sudo ufw allow 8434`

You can check the postgres connection from nodes:

`$ psql -d postgres -h 10.0.2.3 -p 5432 -U postgres`
`$ psql -d postgres -h 10.0.2.4 -p 5432 -U postgres`

Also you can check, if you are able to connect via Haproxy to postgres:  
`$ psql -d postgres -h 10.0.2.3 -p 8432 -U postgres`  
`$ psql -d postgres -h 10.0.2.4 -p 8434 -U postgres`

We have installed Haproxy, but if we disconnect the pi from internet, it will be problem. Because we are configuring
rails database config to write to one of this master host. So if some pi will go offline, and it will be the master pi
on ip 10.0.2.3, we will have problem and will need to change the IP in db config.

I do not know how to solve it via another way. Thanks to Patroni, Etcd and Spilo, we will have good backup, so the only thing  
we will need to change is change port forwarding in router to another ip and change rails database.yml.

Yes, it is not true high availability, but we have only 2 nodes, not three, as is recommended.

## Traefic

On node1:  
`$ sudo mkdir /data/traefic`  
`$ sudo ufw allow 8080`  
`$ sudo ufw allow 80`

On both nodes -
`$ sudo ufw allow 2375`

Overview tutorial is located at [https://docs.traefik.io/v2.2/providers/overview/](https://docs.traefik.io/v2.2/providers/overview/)
To test Traefik, add to your notebook, not to Raspberry pi this configs:

`$ sudo vim /etc/hosts`
```text
10.0.2.3        docker.matho.sk
```

When you enter `http://10.0.2.3:8080` you should see Traefic dashboard.
If you want to host pgadmin on subdomain via Traefik, note, that in label config
`- traefik.docker.network=spilo_db-nw`
`
you need to enter prefix to db-new network. If you deploy your stack with spilo name, the real network name
will be prefixed with spilo_ keyword.

By default, your Traefik admin is publicly available - without some type of basic auth. You can generate
admin:password for example on the page [https://hostingcanada.org/htpasswd-generator/](https://hostingcanada.org/htpasswd-generator/) . Insert username and password, select
apache specific format and copy the credentials to traefik label. Note: escape dolar symbols. For reach dolar, prepend another one dolar:
`- "traefik.http.middlewares.test-auth.basicauth.users=admin:$$apr1$$hlbs$$RQuxjJckwdEefErFO4d3m/"`

To simulate connection to traefic subdomain, add on your notebook, not rpi, following record:
```text
10.0.2.3       traefik.docker.matho.sk
```


## Security
We need to change default Patroni password. Open:
`$ sudo vim /data/my_spilo/spilo1/postgres.yml`  
scroll down and rewrite standby and postgres password. Do it on both of nodes

Also change postgres config at the bottom of the file, to reflect this config:
```text
- local   all             all                                   trust
- host    all             all                127.0.0.1/32       md5
- host    all             all                0.0.0.0/0          md5
- hostssl replication     standby            all                md5
- hostssl all             all                all                md5
```

We dropped row with reject setting, to be able do non https request via our pgadmin. It is not secure, but working for now.
To detect, if postgres started corectly, check `http://10.0.2.3:8008` api endpoint.

If started, check via command line:
`$ psql -d postgres -h 10.0.2.3 -p 5432 -U postgres`

Then check if haproxy is working:
`$ psql -d postgres -h 10.0.2.3 -p 8432 -U postgres`  
`$ psql -d postgres -h 10.0.2.4 -p 8434 -U postgres`


## Rails project import
We need to migrate old rails project images to new rpi.

From your old rpi, execute:  
`$ pg_dumpall -U postgres -h localhost -p 5432 --file=2020_07_07_pg_cluster.sql`  
This command will backup the whole old postgres cluster.

Now you need to login to new rpi, node1. Add your ssh key from old rpi to new node1:
`$ sudo vim ~/.ssh/authorized_keys`

Copy the backup file (execute from old rpi):  
`$ scp -P 7777 2020_07_07_pg_cluster.sql ubuntu@10.0.2.3:~/`

If you have setup correctly volumes, you do not need to copy it to docker container, but copy it to your volume folder, instead:  
`$ sudo cp 2020_07_07_pg_cluster.sql /data/my_spilo/spilo1/2020_07_07_pg_cluster.sql`

I had issues with postgres role in backup file, it wrote, that it already exists. So I need manually open the postgres backup file
and comment with `--` the create postgres role rows

Then I have another problem, with locale `sk_SK.UTF-8`. If I run `$ locale -a` in spilo container, it didnt contain
`sk_SK.UTF-8` locale. So I manually open the postgres backup file, again, and replace `sk_SK.UTF-8` with `en_US.utf8`

Then, you should be able to run:  
`$ psql -U postgres -h 10.0.2.3 -p 5432 -f 2020_07_07_pg_cluster.sql`


## Migrate rails projects
Lets create another docker stack.
To be able start this stack, you need to create volumes folders on host.

`$ sudo mkdir -p /storage-pool/data/gymplacisk`
`$ sudo chmod 777 /storage-pool/data/gymplacisk`  
`$ sudo docker login registry.gitlab.matho.sk:5005 -u root -p token`  
`$ sudo docker pull registry.gitlab.matho.sk:5005/mathove-projekty/gymplaci.sk:latest`

`$ sudo mkdir -p /storage-pool/data/mathosk`
`$ sudo chmod 777 /storage-pool/data/mathosk`  
`$ sudo docker pull registry.gitlab.matho.sk:5005/mathove-projekty/matho:latest`

`$ sudo mkdir -p /storage-pool/data/ecavkalnicaeu`
`$ sudo chmod 777 /storage-pool/data/ecavkalnicaeu`  
`$ sudo docker pull registry.gitlab.matho.sk:5005/mathove-projekty/ecavkalnica:latest`

`$ sudo mkdir -p /storage-pool/data/msdc`
`$ sudo chmod 777 /storage-pool/data/msdc`  
`$ sudo docker pull registry.gitlab.matho.sk:5005/root/msdc:latest`

For arm64 + amd64 devices, use:
```yaml
version: "3.6"
 
services:
  gymplacisk:
    image: registry.gitlab.matho.sk:5005/mathove-projekty/gymplaci.sk:latest
    networks:
      spilo_db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "7082:80"
    volumes:
      - "/storage-pool/data/gymplacisk:/app/shared"
    environment:
      - SMTP_ADDRESS=
      - SMTP_DOMAIN=gymplaci.sk
      - SMTP_USERNAME=
      - SMTP_PASSWORD=
      - POSTGRES_HOST=10.0.2.3
      - POSTGRES_DB=gymplaci_production
      - POSTGRES_USER=
      - POSTGRES_PASSWORD=
      - POSTGRES_PORT=8432
      - RAILS_ENV=production
      - RAILS_WORKERS=2
      - RAILS_MIN_THREADS=5
      - RAILS_MAX_THREADS=5 
    deploy:
      labels:      
        - traefik.http.routers.gymplacisk.rule=Host(`gymplaci.sk`, `www.gymplaci.sk`) 
        - traefik.http.services.gymplacisk-service.loadbalancer.server.port=80    
        - traefik.docker.network=spilo_db-nw   
#        - traefik.http.routers.gymplacisk.tls=true
#        - traefik.http.routers.gymplacisk.tls.certresolver=le     
      replicas: 1
      
  mathosk:
    image: registry.gitlab.matho.sk:5005/mathove-projekty/matho:latest
    networks:
      spilo_db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "7083:80"
    volumes:
      - "/storage-pool/data/mathosk:/app/shared"
    environment:
      - SMTP_ADDRESS=
      - SMTP_DOMAIN=matho.sk
      - SMTP_USERNAME=
      - SMTP_PASSWORD=
        
      - POSTGRES_HOST=10.0.2.3
      - POSTGRES_DB=matho_production
      - POSTGRES_USER=
      - POSTGRES_PASSWORD=
      - POSTGRES_PORT=8432
        
      - RAILS_WORKERS=2
      - RAILS_MIN_THREADS=5
      - RAILS_MAX_THREADS=5    
      - RAILS_ENV=production
    deploy:
      labels:      
        - traefik.http.routers.mathosk.rule=Host(`matho.sk`, `www.matho.sk`) 
        - traefik.http.services.mathosk-service.loadbalancer.server.port=80    
        - traefik.docker.network=spilo_db-nw       
      replicas: 1 
      
  ecavkalnicaeu:
    image: registry.gitlab.matho.sk:5005/mathove-projekty/ecavkalnica:latest
    networks:
      spilo_db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "7084:80"
    volumes:
      - "/storage-pool/data/ecavkalnicaeu:/app/shared"
    environment:
      - SMTP_ADDRESS=
      - SMTP_DOMAIN=matho.sk
      - SMTP_USERNAME=
      - SMTP_PASSWORD=
        
      - POSTGRES_HOST=10.0.2.3
      - POSTGRES_DB=ecavbeckov_production
      - POSTGRES_USER=
      - POSTGRES_PASSWORD=
      - POSTGRES_PORT=8432
        
      - RAILS_WORKERS=2
      - RAILS_MIN_THREADS=5
      - RAILS_MAX_THREADS=5    
      - RAILS_ENV=production
    deploy:
      labels:      
        - traefik.http.routers.ecavkalnica.rule=Host(`ecavkalnica.eu`, `www.ecavkalnica.eu`) 
        - traefik.http.services.ecavkalnica-service.loadbalancer.server.port=80    
        - traefik.docker.network=spilo_db-nw       
      replicas: 1   
      
  msdc:
    image: registry.gitlab.matho.sk:5005/root/msdc:latest
    networks:
      spilo_db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "7085:80"
    volumes:
      - "/storage-pool/data/msdc:/app/shared"
    environment:
      - SMTP_ADDRESS=s
      - SMTP_DOMAIN=matho.sk
      - SMTP_USERNAME=
      - SMTP_PASSWORD=
        
      - POSTGRES_HOST=10.0.2.3
      - POSTGRES_DB=msdc_production
      - POSTGRES_USER=
      - POSTGRES_PASSWORD=
      - POSTGRES_PORT=8432
        
      - RAILS_WORKERS=2
      - RAILS_MIN_THREADS=5
      - RAILS_MAX_THREADS=5    
      - RAILS_ENV=production
    deploy:
      labels:      
        - traefik.http.routers.msdc.rule=Host(`moravskoslovenskydhcup.eu`, `www.moravskoslovenskydhcup.eu`) 
        - traefik.http.services.msdc-service.loadbalancer.server.port=80    
        - traefik.docker.network=spilo_db-nw       
      replicas: 1            
      
networks:
  spilo_db-nw:
    external: true      
```

For arm64 + arm64 devices, use:
```yaml
version: "3.6"
 
services:
  gymplacisk:
    image: registry.gitlab.matho.sk:5005/mathove-projekty/gymplaci.sk:latest
    networks:
      - spilo_db-nw    
    ports:
      - "7082:80"
    volumes:
      - "/storage-pool/data/gymplacisk:/app/shared"
    environment:
      - SMTP_ADDRESS=
      - SMTP_DOMAIN=gymplaci.sk
      - SMTP_USERNAME=
      - SMTP_PASSWORD=
      - POSTGRES_HOST=10.0.2.3
      - POSTGRES_DB=gymplaci_production
      - POSTGRES_USER=
      - POSTGRES_PASSWORD=
      - POSTGRES_PORT=8432
      - RAILS_ENV=production
      - RAILS_WORKERS=2
      - RAILS_MIN_THREADS=5
      - RAILS_MAX_THREADS=5 
    deploy:
      labels:      
        - traefik.http.routers.gymplacisk.rule=Host(`gymplaci.sk`, `www.gymplaci.sk`) 
        - traefik.http.services.gymplacisk-service.loadbalancer.server.port=80    
        - traefik.docker.network=spilo_db-nw   
#        - traefik.http.routers.gymplacisk.tls=true
#        - traefik.http.routers.gymplacisk.tls.certresolver=le     
      replicas: 2
      
  mathosk:
    image: registry.gitlab.matho.sk:5005/mathove-projekty/matho:latest
    networks:
      - spilo_db-nw      
    ports:
      - "7083:80"
    volumes:
      - "/storage-pool/data/mathosk:/app/shared"
    environment:
      - SMTP_ADDRESS=
      - SMTP_DOMAIN=matho.sk
      - SMTP_USERNAME=
      - SMTP_PASSWORD=
        
      - POSTGRES_HOST=10.0.2.3
      - POSTGRES_DB=matho_production
      - POSTGRES_USER=
      - POSTGRES_PASSWORD=
      - POSTGRES_PORT=8432
        
      - RAILS_WORKERS=2
      - RAILS_MIN_THREADS=5
      - RAILS_MAX_THREADS=5    
      - RAILS_ENV=production
    deploy:
      labels:      
        - traefik.http.routers.mathosk.rule=Host(`matho.sk`, `www.matho.sk`) 
        - traefik.http.services.mathosk-service.loadbalancer.server.port=80    
        - traefik.docker.network=spilo_db-nw       
      replicas: 2 
      
  ecavkalnicaeu:
    image: registry.gitlab.matho.sk:5005/mathove-projekty/ecavkalnica:latest
    networks:
      - spilo_db-nw      
    ports:
      - "7084:80"
    volumes:
      - "/storage-pool/data/ecavkalnicaeu:/app/shared"
    environment:
      - SMTP_ADDRESS=
      - SMTP_DOMAIN=matho.sk
      - SMTP_USERNAME=
      - SMTP_PASSWORD=
        
      - POSTGRES_HOST=10.0.2.3
      - POSTGRES_DB=ecavbeckov_production
      - POSTGRES_USER=
      - POSTGRES_PASSWORD=
      - POSTGRES_PORT=8432
        
      - RAILS_WORKERS=2
      - RAILS_MIN_THREADS=5
      - RAILS_MAX_THREADS=5    
      - RAILS_ENV=production
    deploy:
      labels:      
        - traefik.http.routers.ecavkalnica.rule=Host(`ecavkalnica.eu`, `www.ecavkalnica.eu`) 
        - traefik.http.services.ecavkalnica-service.loadbalancer.server.port=80    
        - traefik.docker.network=spilo_db-nw       
      replicas: 2   
      
  msdc:
    image: registry.gitlab.matho.sk:5005/root/msdc:latest
    networks:
      - spilo_db-nw      
    ports:
      - "7085:80"
    volumes:
      - "/storage-pool/data/msdc:/app/shared"
    environment:
      - SMTP_ADDRESS=
      - SMTP_DOMAIN=matho.sk
      - SMTP_USERNAME=
      - SMTP_PASSWORD=
        
      - POSTGRES_HOST=10.0.2.3
      - POSTGRES_DB=msdc_production
      - POSTGRES_USER=
      - POSTGRES_PASSWORD=
      - POSTGRES_PORT=8432
        
      - RAILS_WORKERS=2
      - RAILS_MIN_THREADS=5
      - RAILS_MAX_THREADS=5    
      - RAILS_ENV=production
    deploy:
      labels:      
        - traefik.http.routers.msdc.rule=Host(`moravskoslovenskydhcup.eu`, `www.moravskoslovenskydhcup.eu`) 
        - traefik.http.services.msdc-service.loadbalancer.server.port=80    
        - traefik.docker.network=spilo_db-nw       
      replicas: 2            
      
networks:
  spilo_db-nw:
    external: true      
```

Deploy stack via:  
`$ sudo docker stack deploy --compose-file rails-projects-compose.yml rails-projects`

On your notebook, add:  
`$ sudo vim /etc/hosts`

```text
10.0.2.3       gymplacisk.docker.matho.sk
10.0.2.3       mathosk.docker.matho.sk
10.0.2.3       ecavkalnica.docker.matho.sk
10.0.2.3       msdc.docker.matho.sk
10.0.2.3       gitlab.docker.matho.sk
```

One of our images is running on old rails v4 and is not compatible with PostgreSQL, by default. So we need to introduce
to project this monkey-patch [https://stackoverflow.com/questions/58763542/pginvalidparametervalue-error-invalid-value-for-parameter-client-min-messag](https://stackoverflow.com/questions/58763542/pginvalidparametervalue-error-invalid-value-for-parameter-client-min-messag)
Put it for example to initiazers folder.

Then go to node1, clone the repo and try to build it via:
`$ sudo docker build --build-arg RAILS_ENV=production -t registry.gitlab.matho.sk:5005/root/msdc:latest .`

Then push it via:
`$ sudo docker push registry.gitlab.matho.sk:5005/root/msdc:latest`

And try to redownload
`$ sudo docker pull registry.gitlab.matho.sk:5005/root/msdc:latest`

Rerun docker stack via:
`$ sudo docker stack deploy --compose-file rails-projects-compose.yml rails-projects`  
and this time, the msdc project should start correctly.

Now we need to reupload assets (video + images) from old rpi:
`$ sudo chmod 777 /storage-pool/data/gymplacisk/`

First, clean any files in gymplacisk, mathosk folder ..

From old rpi:  
`$ scp -r -P 7777 /data/gymplaci.sk/* ubuntu@10.0.2.3:/storage-pool/data/gymplacisk`  
`$ scp -r -P 7777 /data/matho.sk/* ubuntu@10.0.2.3:/storage-pool/data/mathosk`  
`$ scp -r -P 7777 /data/ecavkalnica/* ubuntu@10.0.2.3:/storage-pool/data/ecavkalnicaeu`  
`$ scp -r -P 7777 /data/moravskoslovenskydhcup.eu/* ubuntu@10.0.2.3:/storage-pool/data/msdc`


## Gitlab
The Gitlab will be stored in another stack -to do not restart it during redeploys of rails stack

`$ sudo mkdir -p /data/gitlab/config`
`$ sudo chmod 777 /data/gitlab/config`

`$ sudo mkdir -p /data/gitlab/logs`
`$ sudo chmod 777 /data/gitlab/logs`

`$ sudo mkdir -p /data/gitlab/data`  
`$ sudo chmod 777 /data/gitlab/data`


```yaml
version: "3.6"
 
services:
  gitlab:
    image: ulm0/gitlab:12.9.2
    networks:
      spilo_db-nw:
        ipv4_address: 10.0.2.3
    ports:
      - "7086:80" 
      - "4431:443"
      - "22:22"
      - "5005:5005"
      - "5000:5000"      
    volumes:
      - /data/gitlab/config:/etc/gitlab
      - /data/gitlab/logs:/var/log/gitlab
      - /data/gitlab/data:/var/opt/gitlab      
    deploy:
      placement:
        constraints:
          - node.role == manager      
      labels:    
        - "traefik.http.routers.gitlab.entrypoints=web"
        - "traefik.http.routers.gitlab.priority=1"
        - "traefik.http.routers.gitlab.rule=Host(`gitlab.matho.sk`)"
        - "traefik.http.routers.gitlab.service=gitlab@docker"
        - "traefik.http.services.gitlab.loadbalancer.server.port=80"

        - "traefik.tcp.routers.gitlab.entrypoints=web-secure"
        - "traefik.tcp.routers.gitlab.rule=HostSNI(`gitlab.matho.sk`)"
        - "traefik.tcp.routers.gitlab.service=gitlab@docker"
        - "traefik.tcp.routers.gitlab.tls.passthrough=true"
        - "traefik.tcp.services.gitlab.loadbalancer.server.port=443"      
        - "traefik.docker.network=spilo_db-nw"  
               
      replicas: 1
      
networks:
  spilo_db-nw:
    external: true      
```

`$ sudo docker pull ulm0/gitlab:12.9.2`

Deploy stack via:  
`$ sudo docker stack deploy --compose-file gitlab-compose.yml gitlab`

No we need to backup and restore data from Gitlab. I tried many cases, trying to scp files from Gitlab data dir,
trying to tar it to keep permissions and ownership, but all the time it failed. When Gitlab restart its container,
all data were deleted. So I have found official backup/restore solution from the tutorial at [https://docs.gitlab.com/ee/raketasks/backup_restore.html](https://docs.gitlab.com/ee/raketasks/backup_restore.html)

The problem was, that I trying to setup volumes to GlusterFS directory. Seems, that because of it it was failing and
the service was restarting all the time. I recommend to set volumes to somewhere outside of GluserFS. Then prepare clean data dir,
and start the gitlab service. It should correctly start, but also there could be some problem with starting.

If the Gitlab do not start correctly, and is restarting all the time, lets do this:
```
sudo docker run -d --privileged \
--hostname gitlab.matho.sk \
--volume /data/gitlab/config:/etc/gitlab \
--volume /data/gitlab/logs:/var/log/gitlab \
--volume /data/gitlab/data:/var/opt/gitlab  \
ulm0/gitlab:12.9.2  tail -f /dev/null
```

When you run this as container, it will not restart all the time. Then log in to container via:
`$ sudo docker container exec -it <container id> bash`

When you are in container, run :
`$ /opt/gitlab/embedded/bin/runsvdir-start &`  
`$ gitlab-ctl reconfigure`

Copy the tar file:  
`$ sudo cp ~/1594226977_2020_07_08_12.9.2_gitlab_backup.tar /data/gitlab/data/backups/1594226977_2020_07_08_12.9.2_gitlab_backup.tar`

Change ownership:
`$ chown git:git /var/opt/gitlab/backups/1594226977_2020_07_08_12.9.2_gitlab_backup.tar`

Stop some services:
`$ gitlab-ctl stop unicorn`
`$ gitlab-ctl stop puma`  
`$ gitlab-ctl stop sidekiq`  
`$ gitlab-ctl status`

Run backup:
`$ gitlab-backup restore BACKUP=1594226977_2020_07_08_12.9.2`

Now you need to sync configs:
`$ sudo mv /data/gitlab/config/ /data/gitlab/config_new`  
`$ sudo tar --same-owner -pxvf ~/gitlab_config.tar.gz -C /data/gitlab/`  
`$ sudo mv /data/gitlab/srv/gitlab/config/ /data/gitlab/config`

Run reconfigure
`$ gitlab-ctl reconfigure`
`$ gitlab-ctl restart`

I do not know, what I did wrong, but seems I have lost all docker images stored in Gitlab registry. Seems, the fastest
way for me is to repush it, since I have downloaded all of them on node1 or node2. Take it into account before you destroy all the data on your old production micro SD card.

`$ sudo docker login registry.gitlab.matho.sk:5005 -u root -p token`  
`$ sudo docker push registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_aarm64`
Do repush of all your images, via similar way.

## Prometheus
I did it a try, but seems, I will not use it. But if you want, here is sample configuraion

On node1:
`$ sudo vim /data/prometheus_1/prometheus.yml`

```text
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```
##  Summary
Hope, you like this tutorial and I helped you a little bit :) Also there is some todo stack, which I keep at the end of this tutorial. If I find time, I will try to improve
this tutorial continuously.

## TODO
- upload GluserFS data to some cloud, like DigitalOcean? If all my rpis will die to have some backup in cloud?
- postgres backups to GlusterFS
- scale Portainer to 2 nodes?
- set ssl for Portainer and add DNS name, subdomain
- mount pgadmin to my domain
- inspect, how cookies (sesions) are saved, if user is not logged out between 2 requests to different node
- do heavy testing of etcd, multiple situations
- attach some images from Patroni, maybe architecture diagram
- recheck all opened ports, if it needs to be opened
- configure ssl for all rails websites, rebuild rails images
- create own stack for Portainer and maybe also for Traefik
- share gitlab data with GlusterFs
- install Monit
- install open vpn
- install Traefik on node2, to be HA, share config via GlusterFS
- allow 5432 port only for nodes <-> communication
- ab benchmark testing, how many requests/s it is able to do
- due to security, rebuild all my rails docker images, fork it from ubuntu 20.04
- add info about router config (which ports needs to add, if you are behind NAT)
- do it correctly start everything after reboot?