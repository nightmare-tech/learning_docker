# What is Docker and its advantages?
Docker is an open platform for developing, shipping, and running applications. Docker enables you to separate your applications from your infrastructure so you can deliver software quickly. With Docker, you can manage your infrastructure in the same ways you manage your applications. By taking advantage of Docker’s methodologies for shipping, testing, and deploying code quickly, you can significantly reduce the delay between writing code and running it in production.


Read more from:
https://docs.docker.com/get-started/overview/


# Installing Docker
https://docs.docker.com/get-docker/


# Listing Current Docker Networks
 ```
┌──(arpit㉿kali)-[~]
└─$ sudo docker network ls                            
NETWORK ID     NAME      DRIVER    SCOPE
d12a8e553c0f   bridge    bridge    local
712b864fbfea   host      host      local
9ed9a3e7aea1   none      null      local
```
# Adding new Docker Containers
`sudo docker run -itd --rm --name thor busybox`
here `-itd`  is used to make it interactable and detached; to make it run in the background
`--rm` is used to make it clean up after itself when we are done with it.
`busybox`  = image's name
`thor`  = container's name

# Getting a shell of a container
`sudo docker exec -it thor sh`

# Listing all Docker Containers
```
┌──(arpit㉿kali)-[~]
└─$ sudo docker ps                                  
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS     NAMES
34335c31b79d   nginx     "/docker-entrypoint.…"   6 minutes ago   Up 6 minutes   80/tcp    example_3
40a7b4e884c9   busybox   "sh"                     7 minutes ago   Up 7 minutes             example_2
ac945dd0e74c   busybox   "sh"                     7 minutes ago   Up 7 minutes             example

```


# Seeing the New network interfaces made by the dockers
```
┌──(arpit㉿kali)-[~]
└─$ bridge link
5: vethe3fcec6@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master docker0 state forwarding priority 32 cost 2 
7: veth93e5403@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master docker0 state forwarding priority 32 cost 2 
9: veth5e21071@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master docker0 state forwarding priority 32 cost 2 

```

## The default network system for Dockers is Bridged.
- It creates virtual access points for the containers; like a switch.
- Hands out ip address to each container. (DHCP)
- The containers are on the same DNS as the host, so they can communicate within themselves; we can ping from one container to the other.
- In the bridged network we cannot access services hosted by a container BY DEFAULT. We have to manually expose these ports.
```
┌──(arpit㉿kali)-[~]
└─$ sudo docker run -itd --rm -p 80:80 --name stormbreaker nginx
```

# Creating your own Network = User-defined Bridge
`sudo docker network create asgard`
where asgard is the name of the network
Why? 
- To isolate the containers from the default network.
- We can ping the containers within the network by NAME as IP Address can change with redeployment of workloads.

### Adding new containers to the new network
```
┌──(arpit㉿kali)-[~]
└─$ sudo docker run -itd --rm --network asgard --name loki busybox
```

![[Pasted image 20220808233014.png]]

# the HOST
When deploying the `nginx`  container to HOST network:
![[Pasted image 20220808233206.png]]
It will share the host's IP address, ports.
Therefore we don't have to expose any ports.

# MACVLAN
![[Pasted image 20220808233624.png]]
They will connect to the access point to which the host is connected to.
They act like virtual machines.
But all containers connect to the same network and can cause problems with router. (Promiscous Mode)
### Creating a MACVLAN network
```
┌──(arpit㉿kali)-[~]
└─$ sudo docker network create -d macvlan \ 
> --subnet 192.168.122.0/24 \
> --gateway 192.168.122.1 \
> -o parent=eth0 \
> newasgard
```
### Connecting a container to MACVLAN network
```
┌──(arpit㉿kali)-[~]
└─$ sudo docker run -itd --rm --network newasgard \
> --ip 192.168.122.5 \
> --name thor busybox
3936c82d871066d978fc67609a2a6076460cd232a803cfd92bf40a2988006683
```


# IPvlan
two mode - **L2** & L3
Allows to connect directly to the network but they allow the host to share its MAC address with the containers.

## Creating an IPvlan network (L2)
**L2 is default**
Its Layer 2 based - MAC addresses, arp responses
```
┌──(arpit㉿kali)-[~]
└─$ sudo docker network create -d ipvlan \
--subnet 192.168.122.0/24 \
--gateway 192.168.122.1 \
-o parent=eth0 \
newasgard
afcc51f91674b0b4135282337cdd0a1a85472e68f2b25db2291ba1cb9a537413
```
## Deploying a container on the network
```
┌──(arpit㉿kali)-[~]
└─$ sudo docker run -itd --rm --network newasgard \
--ip 192.168.122.5 \
--name thor busybox
8029680610ef44165b7db84b32beb2cdd51f1abd863f9491bc5e7e91d7a45afa
```

## Same MAC of the container as the host 
**CONTAINER:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:1f:31:50 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.160/24 brd 192.168.122.255 scope global dynamic noprefixroute eth0
```

**HOST:**
```
┌[arpit@fedora-2] [/dev/pts/2] 
└[~]> arp -a 192.168.122.160
? (192.168.122.160) at 52:54:00:1f:31:50 [ether] on virbr0
```

# Creating an IPvlan Network (L3)
In this the containers will connect to host like its a router.
Its Layer 3 based - IP addresses
![[Pasted image 20220810150944.png]]

- Thor can ping Loki or Mjolnir.
   Basiclly when seprate networks share the same parent interface they can talk to each other.
- But they dont have access to the internet; there is a gateway but *stuff* dosen't know how to find its way back to the container.
- If you want them to access to the internet, we will have to create a static route from the router settings. 
	Since I was working with a virtual network, I didn't knew how to do it. :)

### Creating IPvlan L3
```
                                                                                
┌──(arpit㉿kali)-[~]
└─$ sudo docker network create -d ipvlan \
--subnet 192.168.94.0/24 \
-o parent=eth0 -o ipvlan_mode=l3 \
--subnet 192.168.95.0/24 \
newasgard
40dbca7cac4af986bbce891df1f4b993cfea07910dcfec95b056082396e7b530
```

### Deploying containers on the network
```
┌──(arpit㉿kali)-[~]
└─$ sudo docker run -itd --rm --network newasgard \
--ip 192.168.94.7 \ 
--name thor busybox
35458e303d157836ad07f705cfb088b1c407cfaacde64a261f56da56a749c261
                                                                                
┌──(arpit㉿kali)-[~]
└─$ sudo docker run -itd --rm --network newasgard \
--ip 192.168.94.8 \
--name mjolnir busybox
5ee50a26bf9804bf6bb425f4053cfcf12da9cab21881438294c3b6e94734f5ab
                                                                                
┌──(arpit㉿kali)-[~]
└─$ sudo docker run -itd --rm --network newasgard \
--ip 192.168.95.7 \
--name loki busybox 
2d137c47c82730d64384f4fdbfcd78b7d013b31e0c8fd48f81a2d5320081f446
^[[A                                                                                
┌──(arpit㉿kali)-[~]
└─$ sudo docker run -itd --rm --network newasgard \
--ip 192.168.95.8 \
--name odin busybox 
3d1c9607183aece6fbb281d4656d3f33c0f76813e772cc2e4d2c6e455e8b7f35
```

# Overlay Network
It allows multiple containers running on multiple hosts to be able to talk to each other and remove the complication to allows us to make rules on how those containers would talk to each other.

# None Network
No network at all, we don't have to create it, it is already there.
```terminal
┌──(arpit㉿kali)-[~]
└─$ sudo docker network ls                
NETWORK ID     NAME        DRIVER    SCOPE
749280f1fc76   asgard      bridge    local
11749d389dad   bridge      bridge    local
712b864fbfea   host        host      local
40dbca7cac4a   newasgard   ipvlan    local
9ed9a3e7aea1   none        null      local
```
**Deploying a container in None:**
```terminal
┌──(arpit㉿kali)-[~]
└─$ sudo docker run -itd --rm --network none \     
--name gorr busybox
40efc50326f11ab84a95957eb008e720b3395ec986c2564d29949ce87e31d2d1
                                                                                
┌──(arpit㉿kali)-[~]
└─$ sudo docker exec -it gorr sh              
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
/  
```

---

# Docker Compose
- Installing Docker Compose (for Debian):
	`sudo apt install docker.io docker-compose`


## Making a docker container with `docker-compose`
You should make a seperate folder for each docker-compose thing you are doing because they are going to be with the same name , i.e, `docker-compose`.
- Create a file named `docker-compose.yaml` as docker-compose looks for this name by default.
  Put the following in the `.yaml` file:
```yaml
version: "3"
services:
  website:
    image: nginx
    ports:
      - "8081:80"
    restart: always
```
Run `sudo docker-compose up -d` in the terminal to initialize the container.
OUTPUT:
```terminal
$ docker-compose up -d     
Creating network "coffeetime_default" with the default driver
Creating coffeetime_website_1 ... done
```
So `docker-compose` created the website with the name `coffeetime_website_1` as the name of the directory was `coffeetime`. And it also made a default bridged network and put the container inside.
![[Pasted image 20220924183655.png]]
`docker-compsoe ps` will only show containers made with `docker-compose`
![[Pasted image 20220924183838.png]]

And we can access the website within the network.
![[Pasted image 20220924184030.png]]

##### Stop/Remove all docker-compose created containers
```terminal
┌──(arpit㉿kali)-[~/docker/coffeetime]
└─$ sudo docker-compose down
Stopping coffeetime_website_1 ... done
Removing coffeetime_website_1 ... done
Removing network coffeetime_default
                                                                                      
┌──(arpit㉿kali)-[~/docker/coffeetime]
└─$ sudo docker-compose up -d
Creating network "coffeetime_default" with the default driver
Creating coffeetime_website_1 ... done
                                                                                      
┌──(arpit㉿kali)-[~/docker/coffeetime]
└─$ sudo docker-compose stop 
Stopping coffeetime_website_1 ... done

```

## What is `docker-compose` and its advantage?
as per the [docker documentation](https://docs.docker.com/get-started/08_using_compose/):

[Docker Compose](https://docs.docker.com/compose/) is a tool that was developed to help define and share multi-container applications. With Compose, we can create a YAML file to define the services and with a single command, can spin everything up or tear it all down.

The _big_ advantage of using Compose is you can define your application stack in a file, keep it at the root of your project repo (it’s now version controlled), and easily enable someone else to contribute to your project. Someone would only need to clone your repo and start the compose app. In fact, you might see quite a few projects on GitHub/GitLab doing exactly this now.

## Creating a network with `docker-compose`
Editing the `.yaml` file as the following...
```yaml
version: "3"
services: 
  website:
    image: nginx
    ports:
      - "8081:80"
    restart: always
    networks:
        coffee:
          ipv4_address: 192.168.122.21

networks:
  coffee:
    ipam:
      driver: default
      config:
        - subnet: "102.168.122.0/24"

```
If we run `sudo docker-compose up -d` after we edit the `.yaml` file, accordingly the changes are made.
So in our case it will make the network and put the container in the network.

Similarly we can make more complex `.yaml` files like [these](https://github.com/vulhub/vulhub).
