# Deploying Application Stack to Swarm
## (Docker-Compose + Docker-Swarm+ Stack+ALB)

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

Here is a project in which the application is deployed into a docker swarm in which the application data have been mounted via an NFS server. The basic infrastructure  for this project has been configured as such docker stack deploying the complete application stack to the swarm. The docker swarm is configured with a manager node and 3 worker nodes. 

A compose file has been written  for the PHP and Nginx services. Where the Nginx for serving the application website  and PHP request is served by the PHP-FPM services. Both the containers are provided with the bind mounting volumes. And the same volume part has been mounted with the NFS server for serving the data for the containers. It means, the application will be updated in the NFS server and from there it will be updated to the mounting location in the node, from there it will be updated to all the containers. Since the cluster is created in docker swarm overlay network has been made use for the whole process.

## Features

- Easy to scale up and down the task as per the requirement via the manager.
- Multi-host networking via overlay networking
- Exposing the ports for services to an external load balancer.

## Architecture
![
alt_txt
](https://i.ibb.co/F4GXn2s/stack-8.jpg)

## Pre-Requests
- Docker and docker-compose should be installed.
- Knowledge in Docker, docker-compose, docker-swarm.
- Swarm Manager and worker should be configured. 

## Behind the code

> Update volume path for bind mounting, name, images according to the requirement

```sh
version: '3'
services:

  php:
    image: php:fpm-alpine3.13
    restart: always
    container_name: php
    volumes:
      - /var/nfs:/var/www/html
    networks:
      - mynet
  nginx:
    image: nginx:alpine
    restart: always
    container_name: nginx
    volumes:
      - /var/nfs:/var/www/html/
      - /var/nfs/nginx.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
    networks:
      - mynet

networks:
  mynet:
    driver: overlay
```
- Here the bind-mounted volume as "/var/nfs:/var/www/html/", so the same path /var/nfs have been mounted with the nfs server. 
##### NFS Configuration
- For installing NFS server:  `apt install nfs-kernel-server`
- Update the permissions and ownership of the directory
- Grant NFS share access to Client systems by updating the exports file.

```sh
vim /etc/exports
/var/nfs IP(rw,sync,no_subtree_check)
exportfs -a
systemctl restart nfs-kernel-server
```
- After making the changes as mentioned mount the same path in every system. 

### Deploy Stack to Swarm 
- Initially change the availability to drain for the manager while proceeding.
```sh
docker node update --availability=drain <Manager_IP_Address>
```

- Deploying  stack to swarm, as the stack command supports compose file. Initially created a registry to distribute images (either can use the Docker hub). 
```sh
 docker service create --name registry --publish published=5000,target=5000 registry:2
````
- Confirm status with `docker service ls`. Further moves forward with starting the application as we have already prepared the docker-compose.yml file.  For the same execute the command.
```sh
docker-compose up -d
```
- Check that the app is running with "docker-compose ps". After everything is fines, let's bring down the app, by `docker-compose down --vloumes`. 

- Further needs to distribute the app’s image across the swarm, it needs to be pushed to the registry. It can be achieved by `docker-compose push`.

- After the same deploy the stack to the swarm. Here the last argument is the name of the stack and it will be prefixed with each network, volume, and service name.

```sh
docker stack deploy --compose-file docker-compose.yml stackdemo
```
- Confirm the running status by `docker stack services stackdemo`
- If required increase the replica count anytime by `docker service scale stackdemo_nginx=6 stackdemo_php=6`

## Conclusion

Here I have configured an application and deployed using the docker stack to swarm and at the same time, an ALB is configured for the traffic flow.
