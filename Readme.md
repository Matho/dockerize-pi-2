# How to run Rails apps with Docker Swarm, GlusterFS, Traefik, PostgreSQL, Patroni, Etcd, Haproxy, PgBouncer, Portainer in HA on Raspberry Pi 4

Read time: 80 minutes

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

You need to automount glusterfs on system boot. On the first node, write to fstab
```
$ sudo vim /etc/fstab
gluster1.example.com:/volume1 /storage-pool glusterfs defaults,_netdev 0 0
```

On second node, write to fstab
```
$ sudo vim /etc/fstab
gluster2.example.com:/volume1 /storage-pool glusterfs defaults,_netdev 0 0
```

Now, when you reboot your device, run `df -h` and you should see /storage-pool mount point on both of your devices.

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
        - traefik.http.middlewares.gymplacisk-redirect-web-secure.redirectscheme.scheme=https
        - traefik.http.routers.gymplacisk.middlewares=gymplacisk-redirect-web-secure
        - traefik.http.routers.gymplacisk.rule=Host(`gymplaci.sk`, `www.gymplaci.sk`)
        - traefik.http.routers.gymplacisk.entrypoints=web
      
        - traefik.http.routers.gymplacisk-secure.rule=Host(`gymplaci.sk`, `www.gymplaci.sk`)
        - traefik.http.routers.gymplacisk-secure.tls.certresolver=le
        - traefik.http.routers.gymplacisk-secure.tls=true
        - traefik.http.routers.gymplacisk-secure.entrypoints=web-secure
        - traefik.http.services.gymplacisk-secure.loadbalancer.server.port=80
        - traefik.docker.network=spilo_db-nw  
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
        - traefik.http.middlewares.mathosk-redirect-web-secure.redirectscheme.scheme=https
        - traefik.http.routers.mathosk.middlewares=mathosk-redirect-web-secure
        - traefik.http.routers.mathosk.rule=Host(`matho.sk`, `www.matho.sk`)
        - traefik.http.routers.mathosk.entrypoints=web
      
        - traefik.http.routers.mathosk-secure.rule=Host(`matho.sk`, `www.matho.sk`)
        - traefik.http.routers.mathosk-secure.tls.certresolver=le
        - traefik.http.routers.mathosk-secure.tls=true
        - traefik.http.routers.mathosk-secure.entrypoints=web-secure
        - traefik.http.services.mathosk-secure.loadbalancer.server.port=80
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
        - traefik.http.middlewares.ecavkalnicaeu-redirect-web-secure.redirectscheme.scheme=https
        - traefik.http.routers.ecavkalnicaeu.middlewares=ecavkalnicaeu-redirect-web-secure
        - traefik.http.routers.ecavkalnicaeu.rule=Host(`ecavkalnica.eu`, `www.ecavkalnica.eu`)
        - traefik.http.routers.ecavkalnicaeu.entrypoints=web
      
        - traefik.http.routers.ecavkalnicaeu-secure.rule=Host(`ecavkalnica.eu`, `www.ecavkalnica.eu`)
        - traefik.http.routers.ecavkalnicaeu-secure.tls.certresolver=le
        - traefik.http.routers.ecavkalnicaeu-secure.tls=true
        - traefik.http.routers.ecavkalnicaeu-secure.entrypoints=web-secure
        - traefik.http.services.ecavkalnicaeu-secure.loadbalancer.server.port=80
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
        - traefik.http.middlewares.msdc-redirect-web-secure.redirectscheme.scheme=https
        - traefik.http.routers.msdc.middlewares=msdc-redirect-web-secure
        - traefik.http.routers.msdc.rule=Host(`moravskoslovenskydhcup.eu`, `www.moravskoslovenskydhcup.eu`)
        - traefik.http.routers.msdc.entrypoints=web
      
        - traefik.http.routers.msdc-secure.rule=Host(`moravskoslovenskydhcup.eu`, `www.moravskoslovenskydhcup.eu`)
        - traefik.http.routers.msdc-secure.tls.certresolver=le
        - traefik.http.routers.msdc-secure.tls=true
        - traefik.http.routers.msdc-secure.entrypoints=web-secure
        - traefik.http.services.msdc-secure.loadbalancer.server.port=80
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
## Deployment

If you are working on amd64 machine (with high probability you are), then you need to build rpi images on the rpi. It is possible to build also via qemu project on amd64, but it is slower
due to emulation layer.

So create directory for docker image build on rpi, node1:  
`$ mkdir ~/docker-builds`

Add ssh keys to your Gitlab, and clone repo you want to build.

Build the project:  
`$ sudo docker build --build-arg RAILS_ENV=production -t registry.gitlab.matho.sk:5005/mathove-projekty/matho:latest .`

If build passed, login, if you are not already:  
`$ sudo docker login registry.gitlab.matho.sk:5005 -u root -p token`

Before the push, it is recommended to tag the release and not use the latest tag. We will use the latest tag  
`$ sudo docker push registry.gitlab.matho.sk:5005/mathove-projekty/matho:latest`

Login to second node and do pull (I'm not sure, if it needs to be executed, or the docker pull the image automatically):  
`$ sudo docker pull registry.gitlab.matho.sk:5005/mathove-projekty/matho:latest`

Remove the project container in Portainer and deploy the rails project:  
`$ sudo docker stack deploy --compose-file rails-projects-compose.yml rails-projects`

Note: If you was previously running on Raspbian OS, like me, you need to rebuild your rpi based image to aarch64 architecture.  
If not, you will get error running docker build, but the old images will be working ok, untill the first build.

Note2: If you are running ruby < v2.3 apps, you will have issues to compile ruby with openssl on Ubuntu 20.04. So I used Ubuntu 16.04 for ruby 2.2 apps. I know, it is not secure, but it works.
Also you will have errors with libv v3.16 gems and therubyracer. I was unable to compile the v8 on aarch64, and I strongly recommend, do not try it! You spend a lot of time. It is a lot easier to swap therubyracer gem with
mini_racer and new libv8 version. Also you need to change execjs version to some new, to support mini_racer gem.

Attention! Only some libv8 versions supports aarch64. Check on rubygems.org which one, and compare with the acceptable mini_racer version. If you are running ruby 2.2 apps, I recommend libv8 v5.9.211.38.1 (exactly this !) and mini_racer v0.1.14.  
You dont have many possibilities. This combination works for me.

## Migrate from Gitlab

During the half of year of my usage, the Gitlab certificate has expired multiple times. I had a lot of issues
with renewal of the cert and I end up with setup, where I'm unable to renew cert for Gitlab registry.
So I have decided to migrate all my projects from self-hosted Gitlab on rpi to Gitlab, which is provided
as benefit from Websupport hosting company. Also, it is better to have projects located outside of my cluster networks and
on safe space. The limit of projects in Websupport Gitlab is set to 30. The problem is, that the registry feature is not activated.
So I need another solution for hosting my private docker build images.

I don't need gui for the builded images, so I have decided to use the `registry` project.

Run the command
```
$ sudo docker info | grep 'Insecure' -B 5 -A 5
```

If you dont have setup the domain, on both nodes do:
```
$ sudo vim /etc/hosts 
10.0.2.3 registry.docker.matho.sk
```

10.0.2.3 is private IP of my master node.

Then allow connections on both nodes:
```
$ sudo ufw allow 6000
```

And setup the domain:
```
$ sudo vim /etc/docker/daemon.json 
{
"insecure-registries" : ["127.0.0.0/8", "registry.docker.matho.sk:6000"]
}
```
Then
```
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
$ sudo docker info | grep 'Insecure' -B 5 -A 5
```

In the last command, you should see the registry subdomain in the output

Start the registry project:
```
$ sudo docker run -d -p 6000:5000 --name registry registry:2
```
Then we will try, if the registry works by pull and push some image:
```
$ sudo docker pull ubuntu
$ sudo docker image tag ubuntu registry.docker.matho.sk:6000/ubuntu:latest
$ sudo docker push registry.docker.matho.sk:6000/ubuntu:latest
$ sudo docker pull registry.docker.matho.sk:6000/ubuntu:latest
```
If everything works, we can continue.

Because I'm changing the url of Gitlab, I needed to rename src in Gemfile in few of my projects.
Then, I needed to rebuild the images with new changes and push new images to registry.

For example, for ecav project:
```
$ sudo docker build -t registry.docker.matho.sk:6000/rpi-ruby-2.4-ubuntu-aarch64:latest .
$ sudo docker push registry.docker.matho.sk:6000/rpi-ruby-2.4-ubuntu-aarch64:latest

$ sudo docker build --build-arg RAILS_ENV=production -t registry.docker.matho.sk:6000/ecavkalnica:latest .
$ sudo docker push registry.docker.matho.sk:6000/ecavkalnica:latest
```

I also needed to rebuild Spilo image:
```
$ sudo docker image tag registry.gitlab.matho.sk:5005/root/spilo:v1.6-p3_aarm64 registry.docker.matho.sk:6000/spilo:v1.6-p3_aarm64
$ sudo docker push registry.docker.matho.sk:6000/spilo:v1.6-p3_aarm64
```

Redeploy of new images:
```
$ sudo docker stack deploy --compose-file rails-projects-compose.yml rails-projects
$ sudo docker stack deploy --compose-file gitlab-compose.yml gitlab
$ sudo docker stack deploy --compose-file spilo-compose.yml spilo
```
Then I have removed old containers, old images and check on 10.0.2.4 node, if new image with port 6000
was downloaded and started.

Attention! The port 6000 is not secured. Ensure, you have implemented some autentification between your nodes.  
Maybe this article? [https://medium.com/@cnadeau_/private-docker-registry-part-2-lets-add-basic-authentication-6a22e5cd459b](https://medium.com/@cnadeau_/private-docker-registry-part-2-lets-add-basic-authentication-6a22e5cd459b)

## Postgres backuping

We are running in HA, so if one of the node will became corrupted, we have second node with the current
postgres snapshot. But, what if something breaks, and both nodes will be corrupted? What if we will
need to restore some older version of Postgres backup?

So, I have implemented backuping on daily base. The script will backup the databases to separate file,
and at the specified day. The backups will be rotated on weekly base.

I tried to setup WAL-g project , but I have some issues with it. Because my Postgres cluster size is small -
the backups has ~ 4MB, so I decided to use pg_dump to take snapshots of the database.

I'm doing the backup on both nodes, and files are then stored to Gluster, which will be geo-replicated to another server.

Because we want to use pg_dump utility and we don't have installed Postgres on our nodes (Postgres is in docker container),
we need to install Postgres client package on both of nodes:
```
$ sudo apt-get install postgresql-client
```

This will not install the whole Postgres, but only som support commands. Then we can try to use pg_dumpall command
to check, if the backup works. For 10.0.2.3 replace the private IP of your master node.
```
$ pg_dumpall -h 10.0.2.3 -U postgres > 2020_12_25_cluster_2.sql
```
To do some advanced backup, I'm using scripts from this page [https://wiki.postgresql.org/wiki/Automated_Backup_on_Linux](https://wiki.postgresql.org/wiki/Automated_Backup_on_Linux)
I have copied `pg_backup.config` and `pg_backup_rotated.sh` scripts. Ensure, the `pg_backup_rotated.sh` is runnable (chmod +x)

If you have setup password for Postgres, put before each call of psql or pgdump the new env
`PGPASSWORD="$PASSWORD" `
The PASSWORD env setup in the config file.

Try to run  the script, if it works:
```
$ sudo ./pg_backup_rotated.sh
```

If works, setup crontab:
```
$ sudo crontab -e
0 2 * * * /bin/bash /home/ubuntu/pg_backuping/pg_backup_rotated.sh 
```

## Minio installation

Currently, I have ~50 GB of disk space free. It is a lot of space and I was looking for some kind of
gui to be able upload some files with. The interesting project is Minio.

It is very simple to use. Dont use the docker image, as it would be complicated to read/write to
Gluster volume. It is simple to install without Docker.

Download the aarch64 image
```
$ mkdir minio
$ cd minio
$ wget https://dl.minio.io/server/minio/release/linux-arm64/minio
$ chmod +x minio 
$ sudo cp minio /usr/local/bin/.
$ sudo ufw allow 9200
```

By default, Minio is running on 9000 port. But we are using Portainer on 9000. So changing to 9200

Start the Minio:
```
$ MINIO_ACCESS_KEY=your-access-key MINIO_SECRET_KEY=your-secret-key minio server --address :9200 /storage-pool/
```
Note: Remember the MINIO_ACCESS_KEY and MINIO_SECRET_KEY. This is your credentials to login via GUI

Point to the GUI - open 10.0.2.3:9200 and put your MINIO_ACCESS_KEY and MINIO_SECRET_KEY credentials.
You should be logged in to the Minio. Upload some file, check if the file is in Gluster and also if is replicated
to another node in Gluster cluster.

It is recommended to enable autostart of Minio on system start. Use the guide located at [https://github.com/minio/minio-service/tree/master/linux-systemd](https://github.com/minio/minio-service/tree/master/linux-systemd)

Also, if you want Minio to be available remotely, add port forwarding to your routers.

## Autobackup of GlusterFS to DigitalOcean droplet

GlusterFS contains feature called Geo-Replication. More info about it can be found at [https://medium.com/@msvbhat/distributed-geo-replication-in-glusterfs-ec95f4393c50](https://medium.com/@msvbhat/distributed-geo-replication-in-glusterfs-ec95f4393c50)
Via this feature, you can autobackup your data from Gluster cluster to droplet in another place.

Initially, I have prepared tutorial for Amazon EC2 instance. But the cheapest instance has cost about
$20 per month, so I have migrated the georeplication to DigitalOcean droplet. I have bought the smallest droplet + 50GB volume.

If you want to set the geo-replication for Amazon EC2, simply go to history of this repo, to commit `9d7ad78e70fa9f` and follow the older tutorial.

My cluster consist of 2 nodes with raspberry pi. Initially I thought, that the geo-replication node needs to be
the same architecture. But it works on amd64 too. Also I thought, that the georep node needs to have more
disk space than the master node in cluster. It is not needed. It generates warning before you start georeplication,
but you can force it and it will synchronize also to smaller node, than your master node capacity is.

If you want to start with DigitalOcean, buy the smallest droplet + volume and assign volume to the droplet.

Then you will need to install Gluster. Note: you need to have the exact version of Gluster for georeplication with
the version you are using in cluster. For me, it is v7.6. But the problem is, that since the installation on cluster,
the version in Ubuntu PPA has changed to v7.9 and v7.6 is no more available anywhere. `dpkg-repack` command
will not help you, because in cluster, I'm using different architecture - aarch64. So I needed to build
Gluster from source code.

### Gluster compilation and installation
The build instructions for Gluster on Ubuntu are located at [https://docs.gluster.org/en/latest/Developer-guide/Building-GlusterFS/](https://docs.gluster.org/en/latest/Developer-guide/Building-GlusterFS/)
I'm using Ubuntu 20.04 LTS

Download Gluster source code to your node in DO via
```
$ git clone https://github.com/gluster/glusterfs.git
```

Go to fetched folder and checkout to your required version, for me v7.6
```
$ git checkout v7.6
```

Update system packages
```
$ apt-get install update
```

Install required packages to be able build Gluster
```
$ sudo apt-get install make automake autoconf libtool flex bison  \
    pkg-config libssl-dev libxml2-dev python-dev libaio-dev       \
    libibverbs-dev librdmacm-dev libreadline-dev liblvm2-dev      \
    libglib2.0-dev liburcu-dev libcmocka-dev libsqlite3-dev       \
    libacl1-dev
```

I have installed also this packages, but I'm not sure if it is really required
```
$ apt-get install openssl pkgconf uuid-dev zlib1g zlib1g-dev libtirpc-dev rpcbind
```

In the Gluster folder, run
```
$ ./autogen.sh
```

If it pass, you can run configure command. To enable geo-replication feature, you need to run
```
$ ./configure --enable-gnfs
```

Then you can run
```
$ ./configure
```

If it pass, run
```
$ make
$ make install
```

After installation, I had this error [https://github.com/gluster/glusterfs/issues/1365](https://github.com/gluster/glusterfs/issues/1365)
I have solved it via running
```
$ ldconfig
```
Then I needed to add gluster binaries to PATH.
```
$ vim ~/.bashrc
```

At the end of file add:
```
export PATH=$PATH:/usr/local/sbin
export PATH=$PATH:/usr/local/bin
```

You should be able to restart glusterd via
```
$ systemctl status glusterd.service
```
Enable auto start of Gluster after restart
```
$ $ systemctl enable glusterd.service
```
If Gluster has started, we can continue in georeplication setup

### Gluster georeplication setup

You can mount volume automatically via clicking in DigitalOcean gui. If you want to mount it manually, do on your georep node (replace the volume name with your volume name)
```
$ sudo mkdir -p /mnt/volume_fra1_04
$ sudo mount -o discard,defaults,noatime /dev/disk/by-id/scsi-0DO_Volume_volume-fra1-04 /mnt/volume_fra1_04
$ sudo echo '/dev/disk/by-id/scsi-0DO_Volume_volume-fra1-04 /mnt/volume_fra1_04 ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```
With the last command, you will enable automount of volume after system reboot.

Check, if volume is mounted:
```
$ dh -h
```
```
As of Gluster 3.5 you cannot use a folder as a geo-replication destination.  
As of 3.5 you must replicate from one Gluster volume to another Gluster volume.
```
Create local domain for your Gluster on DO node.
```
$ sudo vim /etc/hosts
```

Add this record. Note: use your private IP address of DO droplet
```
10.135.0.2 gluster-georep.example.com gluster0
```

Create the volume and mount it (change the volume name)
```

$ sudo gluster volume create georep-volume gluster-georep.example.com:/mnt/volume_fra1_04/gluster/geo-replication force
$ sudo gluster volume start georep-volume
$ sudo gluster volume status
$ sudo mkdir /georep-storage-pool
$ sudo mount -t glusterfs gluster-georep.example.com:georep-volume /georep-storage-pool
```

Create the file and check, if it is saved to Gluster volume
```
$ touch /mnt/volume_fra1_04/gluster/geo-replication/hi.txt
$ cat /georep-storage-pool/hi.txt
```
Add to your /etc/fstab automount command for Gluster volume
```
$ sudo vim /etc/fstab
```
and add
```
gluster-georep.example.com:georep-volume /georep-storage-pool glusterfs defaults,_netdev 0 0
```

Reboot your droplet. The disk volume should be automounted and Gluster volume should be auto started.

Now, setup ufw firewall

Allow login to Gluster from my public ip at home:
```
$ sudo apt-get install ufw

$ sudo ufw allow from 87.244.210.58 to any port 24007
$ sudo ufw deny 24007

$ sudo ufw allow from 87.244.210.58 to any port 49152
$ sudo ufw deny 49152

$ sudo ufw allow 22
$ sudo ufw allow 7777
$ sudo ufw allow 24008
```
Later, we will switch from ssh port 22 to 7777. During DO creation phase, I expect you have assigned your ssh public key, so I expect you are able to login already
to your DO droplet via ssh keys.

Now, we should prepare `geoaccount` user. It means, that we will not sync files to root user on georep node, but to geoaccount. The problem is,
that I had issues with running `gluster-mountbroker` command after Gluster was builed. It seems, that the command wasn't build. So I'm skipping this phase for now. If you want to try to setup `gluster-mountbroker` , switch in
this git history for georeplication tutorial with AWS EC2.

Now create the secret pem files. Login to your Gluster master node (NOT DO node) and run:
```
$ gluster-georep-sshkey generate --no-prefix
```
This will generate ssh keys. Print the output
```
$ sudo cat /var/lib/glusterd/geo-replication/common_secret.pem.pub
```
And insert to authorized_keys file in your DO node:
```
$ sudo vim /root/.ssh/authorized_keys
```

Then,  try to login via ssh to your root user from your Gluster master node:
```
$ sudo ssh root@georep.matho.sk -oPasswordAuthentication=no -oStrictHostKeyChecking=no -i /var/lib/glusterd/geo-replication/secret.pem
```
Until now, we haven't changed the ssh port, so use the default 22

On your master node, add the following record to `/etc/hosts`. Insert public IP address of your DO node:
```
$ vim /etc/hosts

207.154.217.167 gluster-georep.example.com
```
I have changed the ssh port from 22 to 7777.
```
$ sudo ufw allow 7777
$ sudo vim /etc/ssh/sshd_config
```
```
Port 7777
```

```
$ sudo systemctl reload sshd
```

Check, if you can login to DO machine via 7777 port.
```
$ sudo ssh root@georep.matho.sk -p 7777 -oPasswordAuthentication=no -oStrictHostKeyChecking=no -i /var/lib/glusterd/geo-replication/secret.pem
```
To be sure, change the port also in config file
```
$ sudo vim  /etc/glusterfs/glusterd.vol
```

Then (not sure if it was needed)
```
$ sudo echo 'SSHD: ALL' >> /etc/hosts.allow
$ sudo gluster volume set georep-volume auth.allow *
```

Create gsyncd.conf file on your master node, based on your volume names:
```
$ sudo mkdir -p /var/lib/glusterd/geo-replication/volume1_georep.matho.sk_georep-volume
$ sudo cp /etc/glusterfs/gsyncd.conf /var/lib/glusterd/geo-replication/volume1_georep.matho.sk_georep-volume/gsyncd.conf
```

Geo-replication is using 'gsyncd’ and it should be located in `/usr/libexec/glusterfs/`.
On Ubuntu this doesn’t exist. So you need to create it and make symbolic link
On master node, run:
```
$ sudo mkdir -p /usr/libexec/glusterfs/     
$ sudo ln -s /usr/lib/aarch64-linux-gnu/glusterfs/gsyncd /usr/libexec/glusterfs/gsyncd 
```

On DigitalOcean node run:
```
$ sudo mkdir -p /usr/libexec/glusterfs/     
$ sudo ln -s /usr/local/libexec/glusterfs/gsyncd /usr/libexec/glusterfs/gsyncd 
```

If you will need log files, the log files are located at (master node)
```
$ sudo cat /var/log/glusterfs/geo-replication/volume1_georep.matho.sk_georep-volume/gsyncd.log
$ sudo tail -n 150 /var/log/glusterfs/geo-replication/cli.log
$ sudo cat /var/log/glusterfs/geo-replication/gverify-slavemnt.log
$ sudo tail -n 150 /var/log/glusterfs/glusterd.log
```

The setting for Gluster are located at:
```
$ sudo cat /etc/glusterfs/glusterd.vol
```

Now try to login to DO node from your master node, via Gluster:
```
$ sudo gluster volume geo-replication volume1 root@georep.matho.sk::georep-volume create ssh-port 7777 push-pem
```

If your disk volume size on DO droplet is less than your master node, it will print error. Don't worry, simply enter force command, it will work
```
$ sudo gluster volume geo-replication volume1 root@georep.matho.sk::georep-volume create ssh-port 7777 push-pem force
```

Start the geo-replication via:
```
$ sudo gluster volume geo-replication volume1 root@georep.matho.sk::georep-volume start
```

See the status via
```
$ sudo gluster volume geo-replication volume1 root@georep.matho.sk::georep-volume status
```

You should see status `Active`

Now, the geo-replication should work.

I have found, that during initial phase, it creates empty folders and zero-byte files and it starts
synchronization after this phase. So do not worry, if you see zero byte length files at the beginning.
It takes some time until the files are synchronized.

## Prometheus + Grafana vs Datadog vs Zabbix

For a long time, I was searching the right solution for the infrastructure monitoring. Datadog is very nice
solution, but the free package is not good enough for me - the alerting/monitoring section is missing
in the free package.

Another solution was Prometheus + Grafana. I give it a try, but the settings seems for me a lot complicated.
So finally, I have decided to use Zabbix.

## Zabix

We will install Zabbix from docker containers. The tutorial is located at [https://www.zabbix.com/documentation/current/manual/installation/containers](https://www.zabbix.com/documentation/current/manual/installation/containers)
Go to source code at [https://github.com/zabbix/zabbix-docker](https://github.com/zabbix/zabbix-docker) and download yaml file
`docker-compose_v3_ubuntu_pgsql_latest.yaml`

To the same folder, where is downloaded this yaml, download dot files like:
- .env_db_pgsql
- .env_srv
- .env_prx
- .env_prx_sqlite3
- .env_db_mysql_proxy
- .env_prx_mysql
- .env_web
- .env_agent
- .env_java
- .env_db_mysql
- .MYSQL_PASSWORD
- .MYSQL_ROOT_PASSWORD
- .MYSQL_USER
- .POSTGRES_PASSWORD
- .POSTGRES_USER

If you start the docker compose now, you will see error `ERROR: no matching manifest for linux/arm64/v8 in the manifest list entries`
It is because that the used Mysql version from Dockerhub does not contain aarch64 image. So change the source image to:

```
mysql-server:
 image: mysql/mysql-server:8.0.22
```

Because we have already assigned ports 80 and 443, change that also:

```
zabbix-web-apache-pgsql:
  image: zabbix/zabbix-web-apache-pgsql:ubuntu-5.2-latest
  ports:
    - "8085:8080"
    - "4435:8443"
```

Do not forget to allow this ports in firewall:
```
$ sudo ufw allow 8085
$ sudo ufw allow 4435
```

Now, Zabbix will be located at `10.0.2.3:8085`  The default credentials are:
```
Admin:zabbix
```

Lets start Zabbix via docker compose (for now, without detached status). If you want to run it in background, add `-d` cmd switch
```
$ sudo docker-compose -f ./docker-compose_v3_ubuntu_pgsql_latest.yaml up
```

This will create docker volume folder in he docker compose file.

Now you need to assign Zabbix host ip in Zabbix settings. Go to Portainer, select zabbix server container and click on details.
Scroll down and you will se available networks, which this container uses. Search for backend network and copy the docker IP.
For me , the IP was `172.16.239.7`

Now go to Zabbix, login via default credentials. Navigate to configuration -> hosts and instead of 127.0.0.1
change the ip to `172.16.239.7`. You should now see at the end of row green word ZBX and set the ip to
`172.16.239.7:10050`

Now your agent is configured and you can monitor your infrastructure.

You can continue in setuping triggers (email triggers) [https://www.zabbix.com/documentation/current/manual/quickstart/trigger](https://www.zabbix.com/documentation/current/manual/quickstart/trigger)

It is good to have sms notifications, in case of some serious issue. I have setup it based on this tutorial [https://github.com/ClickSend/zabbix-sms](https://github.com/ClickSend/zabbix-sms)
The file, I'm using is

```
#!/bin/sh
#
# Script to send SMS alerts via Zabbix
#
# More information about the ClickSend API: https://developers.clicksend.com/docs/http/v2/#ClickSend-v2-API-SMS
#
# These lines are optional (Sending the output to a file)
#
#
exec 3>&1 4>&2
trap 'exec 2>&4 1>&3' 0 1 2 3
exec 1>/dev/null 2>&1

# Your ClickSend credentials (from https://dashboard.clicksend.com/#/account/subaccount)
USERNAME=username
APIKEY=apikey
PHONE=myphone
MESSAGE=${2}

# This echo is only for my /var/run/zabbix/log/log.out script
echo "date +%H:%M:%: Sending SMS Text to $1"

#
# Posting the SMS via HTTP API
# 
# NOTE: Because the Zabbix alerts have spaces/new lines, we are using the "--data-urlencode" option to encode the message.
#
/var/lib/zabbix/modules/curl -X POST "https://api-mapper.clicksend.com/http/v2/send.php?method=zabbix&username=$USERNAME&key=$APIKEY&to=$PHONE" --data-urlencode message="$MESSAGE"

echo "date +%H:%M:%: SMS Text sent to $1"
echo "MESSAGE: $2"
```

Change USERNAME, APIKEY and PHONE variables. I have decided to hardcode PHONE variable in the script, to allways send
sms only to my phone number - for security issues. Also you can find the `/var/lib/zabbix/modules/curl` line.
Because in the configured docker image, there is no curl, I have decided to copy curl to this volume from my native system and it works.
```
$ sudo cp /usr/bin/curl /home/ubuntu/docker/rpi_docker_compose/zbx_env/var/lib/zabbix/modules
```
Register account in ClickSend system, it is ready in a minute. You can go to the list of media types and trigger test sms.

The docker-compose process is not started on system start. So, if system is restarted, it doesn't start the process.
Lets add upstart script
```
$ sudo vim /etc/systemd/system/zabbix.service
$ sudo chmod +x /etc/systemd/system/zabbix.service
```
Insert the text:
```
[Unit]
Description=Docker Compose Application Service
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ubuntu/docker/rpi_docker_compose
ExecStart=/usr/bin/docker-compose -f ./docker-compose_v3_ubuntu_pgsql_latest.yaml up -d
ExecStop=/usr/bin/docker-compose -f ./docker-compose_v3_ubuntu_pgsql_latest.yaml down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

Ensure, you have provided correct directories. Then restart daemon
```
$ sudo systemctl daemon-reload
```

And start the service. Also, start the service on system start
```
$ sudo systemctl start zabbix.service
$ sudo systemctl enable zabbix.service
```

We also need to install agent on the second node.
Lets run:
```
$ sudo docker run --name zabbix-agent -d -p 10050:10050 --restart always --privileged -e ZBX_HOSTNAME="ubuntu2" -e ZBX_SERVER_HOST="10.0.2.3" zabbix/zabbix-agent2:alpine-5.2.3
```

Not sure if needed, but lets open the firewall
```
$ sudo ufw allow 10050
$ sudo ufw allow 10051
```

After it, you need to add new server in Zabbix backend. Go to Configuration > Hosts > Create host and fill in settings for IP 	10.0.2.4
You can check detailed instruction in article [https://techexpert.tips/zabbix/zabbix-monitor-linux-using-agent/](https://techexpert.tips/zabbix/zabbix-monitor-linux-using-agent/) in the section
Tutorial - Zabbix Monitor Linux.

## Using custom certificate with Traefik

I have used this tutorial [https://kevinquillen.com/setting-traefik-2-local-ssl-certificate](https://kevinquillen.com/setting-traefik-2-local-ssl-certificate)

## Gitlab Pipelines

I'm using Gitlab instance, which is hosted in Websupport. Websupport do not support pipeline runners, you need to host runners on your system.

We will install runner on rpi with 4GB ram, as there is not so many services running, like on rpi with 8GB ram.

The page with installation instructions is located at [https://docs.gitlab.com/runner/install/linux-repository.html](https://docs.gitlab.com/runner/install/linux-repository.html)
```
$ curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
$ export GITLAB_RUNNER_DISABLE_SKEL=true; sudo -E apt-get install gitlab-runner
```
Now we need to register runner. Instructions are located at [https://docs.gitlab.com/runner/register/index.html](https://docs.gitlab.com/runner/register/index.html)
I have selected project specific runner.
```
$ sudo gitlab-runner register
```
You need to answer some questions. Navigate in the Gitlab to your project, in left menu to Settings, then to CI/CD and expand the runners section. Now copy the url and token under the `Set up a specific runner manually`
When you are asked for tags, enter: `build,deploy`. Now your runner should be running.

We need to configure the runner. Open
```
$ sudo vim /etc/gitlab-runner/config.toml
```
And change the priviledge to `true`. Under the key value priviledge, add:
```
extra_hosts = ["registry.docker.matho.sk:10.0.2.3"]
```

This means, that the started docker in runner will understand the registry url and will try to connect to 10.0.2.3 IP for docker registry.

Also change the gitlab-runner user permissions:
```
$ sudo usermod -aG docker gitlab-runner 
$ sudo -u gitlab-runner -H docker info 
```
You should see docker info. Because we have changed the configuration, we will need to restart service
```
$ sudo systemctl restart gitlab-runner.service
$ sudo systemctl status gitlab-runner.service
```
Check, if runner has correctly started.

### .gitlab-ci.yml

We have used this config. The file is located in your rails project, in the root `.gitlab-ci.yml`

```yaml
stages:
  - build

build-master:
  stage: build
  image: registry.docker.matho.sk:6000/docker-gitlab-pipelines-aarch64:latest
  tags:
    - build
  variables:
    DEPLOY_ENV: production
    RAILS_ENV: production
  script:
    - ruby ./bin/build.docker.rb
  timeout: 180 minutes
  only:
    - master
```
As you can see, the base image is `docker-gitlab-pipelines-aarch64:latest` We need to create this image first. Create project in Gitlab, name it `docker-gitlab-pipelines-aarch64`
Prepare new Dockerfile inside the project.

```
vim Dockerfile
```
The content is:

```
# Pull base image
FROM ubuntu:20.04 

MAINTAINER Martin Markech <martin.markech@matho.sk>

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
	&& apt-get install -y --no-install-recommends \
		bzip2 \
		ca-certificates \
		curl \
		libffi-dev \
		libssl-dev \
		libyaml-dev \
		procps \
		zlib1g-dev \
                git \                
	&& rm -rf /var/lib/apt/lists/*

RUN apt-get update \ 
    && apt-get install -y \
    apt-transport-https \
    gnupg \
    lsb-release

RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

RUN echo \
  "deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

RUN apt-get update

RUN apt-get install -y docker-ce docker-ce-cli containerd.io

ENV RUBY_MAJOR 2.7
ENV RUBY_VERSION 2.7.2
ENV RUBY_DOWNLOAD_SHA256 6e5706d0d4ee4e1e2f883db9d768586b4d06567debea353c796ec45e8321c3d4 

# some of ruby's build scripts are written in ruby
# we purge this later to make sure our final image uses what we just built
RUN buildDeps=' \
		autoconf \
		bison \
		gcc \
		libbz2-dev \
		libgdbm-dev \
		libglib2.0-dev \
		libncurses-dev \
		libreadline-dev \
		libxml2-dev \
		libxslt-dev \
		make \
		ruby \
	' \
	&& set -x \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends $buildDeps \
	&& rm -rf /var/lib/apt/lists/* \
	&& mkdir -p /usr/src/ruby \
	&& curl -fSL -o ruby.tar.gz "http://cache.ruby-lang.org/pub/ruby/$RUBY_MAJOR/ruby-$RUBY_VERSION.tar.gz" \
	&& echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.gz" | sha256sum -c - \
	&& tar -xzf ruby.tar.gz -C /usr/src/ruby --strip-components=1 \
	&& rm ruby.tar.gz \
	&& cd /usr/src/ruby \
	&& autoconf \
	&& ./configure --disable-install-doc \
	&& make -j"$(nproc)" \
	&& make install \
	&& rm -r /usr/src/ruby \
	&& apt-get purge -y --auto-remove $buildDeps

# skip installing gem documentation
RUN echo 'gem: --no-rdoc --no-ri' >> /.gemrc

# install things globally, for great justice
ENV GEM_HOME /usr/local/bundle
ENV PATH $GEM_HOME/bin:$PATH
RUN gem install bundler \
  && bundle config --global path "$GEM_HOME" \
  && bundle config --global bin "$GEM_HOME/bin"

RUN apt-get update && apt-get install -y systemd

# don't create ".bundle" in all our apps
ENV BUNDLE_APP_CONFIG $GEM_HOME

CMD [ "irb" ]
```
Pull the git repo to rpi. Build and push to docker registry
```
$ sudo docker build -t registry.docker.matho.sk:6000/docker-gitlab-pipelines-aarch64:latest .
$ sudo docker push registry.docker.matho.sk:6000/docker-gitlab-pipelines-aarch64:latest
```

Now we have base image with Docker and Ruby installed inside. We can continue.


Create file `bin/build.docker.rb` in your rails project. You need to provide `gitlab_readonly_token` value. Go to your Gitlab project settings and navigate to section `deploy tokens`. Create new pair and insert in the following code

```ruby
#!/usr/bin/env ruby

# Execute system command or exit if it fails
def execute!(command)
  system(command) || exit(1)
end

current_folder = Time.now.to_i
folder_path = "/home/gitlab-runner/docker-gitlab-builds/gymplaci/#{current_folder}"
# TODO modify this 2 values
gitlab_readonly_user = 'gitlab-deploy-token-1'
gitlab_readonly_token =  "YOUR-PASS-HERE"

execute!( 'mkdir -p /etc/docker')
execute!('touch /etc/docker/daemon.json')
execute!('echo \'{ "insecure-registries": [ "127.0.0.0/8", "registry.docker.matho.sk:6000" ], "storage-driver": "vfs" }\' > /etc/docker/daemon.json')
execute!( 'cat /etc/docker/daemon.json')
execute!( 'service docker stop')
execute!( 'service docker start')
execute!( 'service docker status')
execute!( 'docker info')

execute!("git clone https://#{gitlab_readonly_user}:#{gitlab_readonly_token}@gitlab.websupport.sk/matho/gymplaci.git #{folder_path}")
puts "git clone done"

execute!("docker build --build-arg RAILS_ENV=production -t registry.docker.matho.sk:6000/gymplaci:latest #{folder_path}")
puts "build done"

execute!('docker push registry.docker.matho.sk:6000/gymplaci:latest')
puts "push completed"

puts "FINISHED"

```

Commit your changes. When you push to master branch, the Gitlab pipelines should be started and your new build will be prepared.

Note: We have only automate the build process, not the deploy. The deploy process is little complicated as `docker stack deploy` will not work from inside docker-in-docker (Gitlab pipeline)
So I'm skipping this section. You can deploy manually via cmdline or via Portainer easily.

Note 2: The build process is very slow. It took 105 minutes to build the image. Probably it is because of docker-in-docker and vfs driver. I tried to use overlay2 or overlay or aufs driver, but the
only one which was working was the vfs driver.

##  Summary
Hope, you like this tutorial and I helped you a little bit :) Also there is some todo stack, which I keep at the end of this tutorial. If I find time, I will try to improve
this tutorial continuously.

## TODO
- add info what I need to start after instance is restarted
- scale Portainer to 2 nodes?
- set ssl for Portainer and add DNS name, subdomain
- mount pgadmin to my domain
- inspect, how cookies (sesions) are saved, if user is not logged out between 2 requests to different node
- do heavy testing of etcd, multiple situations
- attach some images from Patroni, maybe architecture diagram
- recheck all opened ports, if it needs to be opened
- create own stack for Portainer and maybe also for Traefik
- share gitlab data with GlusterFs
- install open vpn
- install Traefik on node2, to be HA, share config via GlusterFS
- allow 5432 port only for nodes <-> communication
- ab benchmark testing, how many requests/s it is able to do
- add info about router config (which ports needs to add, if you are behind NAT)
- do it correctly start everything after reboot?
