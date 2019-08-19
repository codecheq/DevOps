# DevOps
DevOps first project
Prerequisites

Must have Docker (Community Edition will suffice) running.
Build war

Build war file to have something to deploy on codech:

mvn clean install

this application uses replicated HTTP session (<distributable/> tag in src/main/webapp/WEB-INF/web.xml).
Build docker image with codech+war

Build a Docker image containig war running inside codech configured to run in clustered mode (-server-config=standalone-ha.xml)

docker build -t codech-cluster-node .

Create a dedicated Docker network

Defining a Docker network specifing gateway and subnet is necessary to assign specific IPs to containers:

docker network ls 
docker network create --driver bridge --gateway 172.19.0.1 --subnet 172.19.0.0/16 cluster_nw  
docker network inspect cluster_nw  

Run a Docker web GUI

docker run --restart=always -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer  

The console is now available at: http://localhost:9000/#/dashboard
Run 2 Docker containers

Run 2 containers.
Note that specific IPs are only needed to configure a mod_jk balancer later.

docker run --network=cluster_nw --ip 172.19.0.2 -d -p 8081:8080 -p 9991:9990 --name codech-cluster-node-1 -t codech-cluster-node  
docker run --network=cluster_nw --ip 172.19.0.3 -d -p 8082:8080 -p 9992:9990 --name codech-cluster-node-2 -t codech-cluster-node

Check clustering is working

http://localhost:8081/codech/helloworld
http://localhost:8082/codech/helloworld
Inspect a Docker container

docker container ls --all  
docker exec -it 24d9244eb9f0 bash

Remove a Docker container

docker container stop 8b64379c60e4  
docker container rm 8b64379c60e4

Load Balancer with mod_jk

go to src/mod_jk folder

docker build -t httpd-mod_jk .

then run

docker run --network=cluster_nw --ip 172.19.0.4 -d -p 80:80 --name mod_jk -t httpd-mod_jk

finally try url http://localhost/codech/helloworld for cluster try url http://localhost:???/mcm for console
Load Balancer with mod_cluster

using https://hub.docker.com/r/codech/mod_cluster-master-dockerhub (GIT https://github.com/codech/mod_cluster-dockerhub):

docker pull codech/mod_cluster-master-dockerhub

build mod_cluster:

cd ./src/mod_cluster  
docker build -t mod_cluster_image . 

build codech:

docker build -f Dockerfile-mod_cluster -t codech-mod_cluster-node .

start httpd with mod_cluster:

docker run -P -i -d --network=cluster_nw --ip 172.19.0.5 -p 80:80 --name mod_cluster mod_cluster_image 

start codech:

docker run --network=cluster_nw --ip 172.19.0.6 -p 8086:8080 -p 9996:9990 --name codech-mod_cluster-node-1 -t codech-mod_cluster-node  
docker run --network=cluster_nw --ip 172.19.0.7 -p 8087:8080 -p 9997:9990 --name codech-mod_cluster-node-2 -t