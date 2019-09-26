# Deployment on a Docker Swarm

Even with only a single node you can create a Docker Swarm. If you have access to additional nodes you can add them and create a cluster, but that is not necessary for the step in this exercise.

## Creation of the Swarm

To enable Swarm on your Docker Engine:

```bash
$ docker swarm init
Swarm initialized: current node (o16nlc33ukub7zel71z3pcas1) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-51zt7i82u7gpyc2avhdbk9os26w6iiz9ltdr9ffcg7vwicem9n-3m9asope502zv9ehb02d2fxns 192.168.65.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

That's it! You've not got a Docker Swarm-enabled node. If you're on a Linux system, play-with-docker.com or play-with-k8s.com you can add additional nodes by copying the `docker swarm join` command you've been given. Again, this is optional for these exercises.

> **NOTE:** You cannot add additional nodes to Docker Desktop.

## Create a DNS load balancer

In order to load balance the traffic towards several instances of our **app** service, we will add a new service. This one uses the DNS round-robin capability of Docker engine (version 1.11+) for containers with the same network alias.

Note: to present the DNS round-robin feature, we do not use the load balancer of the previous chapter (dockercloud/haproxy).

The following Dockerfile uses nginx:1.9 official image and add a custom nginx.conf configuration file. Copy the following into a file named `Dockerfile-nginx`:

```dockerfile
FROM nginx:1.9

# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Copy the following into a file named `nginx.conf` - this is required by nginx to configure load balancing and defines a proxy_pass directive towards **http://apps** for each request received on port 80.

**apps** is the value we will set as the app service network alias.

```json
user nginx;
worker_processes 2;
events {
  worker_connections 1024;
}
http {
  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;

  # 127.0.0.11 is the address of the Docker embedded DNS server
  resolver 127.0.0.11 valid=1s;
  server {
    listen 80;
    # apps is the name of the network alias in Docker
    set $alias "apps";

    location / {
      proxy_pass http://$alias;
    }
  }
}
```

Let's build and publish the image of this new load balancer to Docker Hub. Reminder - substitute your Hub account info in place of `jimmyarms`:

```bash
# Create image
$ docker build -t jimmyarms/lb-dns -f Dockerfile-nginx .
[+] Building 13.2s (9/9) FINISHED

# Publish image
$ docker push -t jimmyarms/lb-dns
```

The image can now be used in our Docker Compose file.

## Update our docker-compose file

The new version of the docker-compose.yml file is the following one

```yaml
version: '3.6'
services:
  mongo:
    image: mongo:4.2
    networks:
      - backend
    volumes:
      - mongo-data:/data/db
    expose:
      - "27017"
  lbapp:
    image: jimmyarms/lb-dns
    networks:
      - backend
    ports:
      - "8000:80"
  app:
    image: jimmyarms/labs-nodejs:001
    expose:
      - "80"
    environment:
      - MONGO_URL=mongodb://mongo/messageApp
    networks:
      backend:
        aliases:
          - apps
    depends_on:
      - lbapp
volumes:
  mongo-data:
networks:
  backend:
    driver: overlay
```

There are several important updates here
* usage of the lb-dns image for the load balancer service
* constraints to choose the nodes on which each service will run (needed in our example to illustrate the DNS round robin)
* creation of a new user-defined overlay network to enable each container to communicate with each other through their name
* for each service, definition of the network used
* definition of network alias for the **app** service (crucial item as this is the one that will enable nginx to proxy requests)

## Deployment and scaling of the application

In order to run the application in this Swarm, we will issue the following commands
* switch to the swarm master context ```eval $(docker-machine env --swarm demo0)```
* run the new compose file ```docker-compose up```
* increase the number of **app** service instances ```docker-compose scale app=5```

Our application is then available through http://192.168.99.101:8000/message

192.168.99.101 is the IP of the Swarm master. 8000 is the port exported by the load balancer to the outside.


