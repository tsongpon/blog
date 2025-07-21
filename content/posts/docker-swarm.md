---
title: Container Orchestration with Docker Swarm
description: Docker Swarm อีกหนึ่งทางเลือกของคนที่อยากมี self-hosted container orchestration platform ไว้ใช้ แต่ไม่อยากจะใช้ Kubernetes ซึ่งมีความซับซ้อน และมี learning curve ที่ค่อนข้างสูงในการ setup และการ operate cluster
date: 2024-06-19
draft: false
tags: [docker,container]
---

Docker Swarm อีกหนึ่งทางเลือกของคนที่อยากมี self-hosted container orchestration platform ไว้ใช้ แต่ไม่อยากจะใช้ Kubernetes ซึ่งมีความซับซ้อน และมี learning curve ที่ค่อนข้างสูงในการ setup และการ operate cluster

ถึงแม้ swarm จะไม่ได้มี feature หรูหรา เทียบเท่า Kubernetes แต่สำหรับทีมขนาดเล็ก และขนาดกลางที่ต้องการเพียงแค่ฟังก์ชั่นพื้นฐานของ container orchestration platform ก็คิดว่าน่าจะเพียงพอกับการใช้งาน

# Setup Docker Swarm Cluster

Requirements
1. Linux host 3 เครื่อง (ในที่นี้จะใช้ RedHat 9 Linux ที่มี IP เป็น 10.0.12.137, 10.0.9.198, 10.0.10.73)
2. ทั้งสามเครื่องจะต้องเปิด port 2377 (TCP), 7946 (TCP/UDP) และ 4789 (UDT)

Create cluster

**ที่เครื่อง `10.0.12.137` (node-1)**

1. Install Docker
```bash
#add repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

#Install docker\
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

#Start docker
sudo systemctl start docker
```

2. Initial swarm cluster

```bash
sudo docker swarm init --advertise-addr 10.0.12.137
```

จะได้ output ออกมาประมาณนี้

```bash
Swarm initialized: current node (hw5dhqf842lkookic1lpt7z4y) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0wecnrittwmr862fzm9h54aoipv24egam1ap5wsbbw34hvl0k7-07lm7ktxvhy5r6xq15mydyjxp 10.0.12.137:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

จาก output เราจะได้คำสั่งสำหรับ join cluster ทีจะเอาไปรันในเครื่อง worker node

```bash
docker swarm join --token SWMTKN-1-0wecnrittwmr862fzm9h54aoipv24egam1ap5wsbbw34hvl0k7-07lm7ktxvhy5r6xq15mydyjxp 10.0.12.137:2377
```

**ที่เครื่อง `10.0.9.198` (node-2)**
1. Install Docker

```bash
#add repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

#Install docker\
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

#Start docker
sudo systemctl start docker
```

2. รันคำสั่ง join cluster ที่ได้จากการ initial cluster จาก master node

```bash
sudo docker swarm join --token SWMTKN-1-0wecnrittwmr862fzm9h54aoipv24egam1ap5wsbbw34hvl0k7-07lm7ktxvhy5r6xq15mydyjxp 10.0.12.137:2377
```

**ที่เครื่อง `10.0.10.73` (worker node-3)**
1. Install Docker

```bash
#add repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

#Install docker\
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

#Start docker
sudo systemctl start docker
```

2. รันคำสั่ง join cluster ที่ได้จากการ initial cluster จาก master node

```bash
sudo docker swarm join --token SWMTKN-1-0wecnrittwmr862fzm9h54aoipv24egam1ap5wsbbw34hvl0k7-07lm7ktxvhy5r6xq15mydyjxp 10.0.12.137:2377
```

เสร็จแล้วกลับไปที่ master node เพื่อดูสถานะของ cluster ที่เราเพิ่งสร้าง

**ที่เครื่อง `10.0.12.137` (node-1)**

```bash
[ec2-user@ip-10-0-12-137 ~]$ sudo docker node ls
ID                            HOSTNAME                                         STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
lc1sjgj9e94wah0rpvtenzwqx     node-3                                            Ready    Active                          26.1.4
2yd6b27g1n0jhcsrgeq16qzan     node-2                                            Ready    Active                          26.1.4
hw5dhqf842lkookic1lpt7z4y *   node-1                                            Ready    Active         Leader           26.1.4
```

ตอนนี้เราได้ Docker Swarm cluster ไว้ใช้งานแล้ว

# ทดลอง deploy service

deploy Nginx ไปที่ cluster และ expose service ที่ port 80

```bash
sudo docker service create --name web --replicas 3 --publish published=80,target=80 nginx:latest
```

ลอง list service ดูด้วยคำสั้ง

```bash
[ec2-user@node-1 ~]$ sudo docker service ls
ID             NAME      MODE         REPLICAS   IMAGE          PORTS
zi2xd9yvuimr   web       replicated   3/3        nginx:latest   *:80->80/tcp
```

ทดลองเรียก service

```bash
[ec2-user@node-1 ~]$ curl 10.0.12.137
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

service up and running :smiley:

# Promote node เป็น manager (Optional)

เราสามารถ promote node เป็น manager (เพื่อที่จะให้เราสามารถรันคำสั่งที่ใช้จัดการเกี่ยวกับ cluster ในหลายๆ node) ได้ด้วยคำสั้ง (สามารถให้ทุก node ใน cluster เป็น manager role ได้)

```bash
[ec2-user@node-2 ~]$ sudo docker node promote node-2
Node node-2 promoted to a manager in the swarm.
```
