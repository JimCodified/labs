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

In order to load balance the traffic towards several instances of our **app** service, we will add a new service. To provide an example that's more realistic than the last exercise we'll use Nginx to do the load balancing, and use the DNS round-robin capability of Docker Engin for containers with the same network alias. Similar to before, this lets us scale our containers up & down without the need to track IP addresses and/or ports.

The following Dockerfile uses nginx:1.9 official image and add a custom nginx.conf configuration file. Copy the following into a file named `Dockerfile-nginx`:

```dockerfile
FROM nginx:1.17.4-alpine

# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Copy the following into a file named `nginx.conf` - this is required by nginx to configure load balancing and defines a proxy_pass directive towards **http://apps** on port 1337 for each request received on port 80.

**apps** is the value we will set as the app service network alias.

```text
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
      proxy_pass http://$alias:1337;
    }
  }
}
```

Let's build and publish the image of this new load balancer to Docker Hub. Reminder - substitute your Hub account info in place of `jimmyarms`:

```bash
# Create image
$ docker build -t jimmyarms/lb-dns:001 -f Dockerfile-nginx .
[+] Building 13.2s (9/9) FINISHED

# Publish image
$ docker push -t jimmyarms/lb-dns:001
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
  lbapp:
    image: jimmyarms/lb-dns:001
    networks:
      - frontend
    ports:
      - "8000:80"
  app:
    image: jimmyarms/labs-nodejs:001
    environment:
      - MONGO_URL=mongodb://mongo/messageApp
    networks:
      frontend:
        aliases:
          - apps
      backend:
    depends_on:
      - lbapp
volumes:
  mongo-data:
networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
```

There are several important updates here

* Usage of the lb-dns image for the load balancer service, which is built on nginx
* Creation of two user-defined overlay networks to enable appropriate container communication
  * Our nginx load balancer can talk to the app service, using the **apps** network alias instead of explicit addresses & ports
  * **apps** will be resolved by the internal Docker Engine DNS server and round robin amongst the services
  * The **app** service can communicate to the **mongo** service on the `backend` network
  * The **lbapp** service CANNOT communicate to the **mongo** service since they are not on the same network

## Deployment and scaling of the application

* In order to run the application in this Swarm, we use the following command:

```bash
$ docker stack deploy -c docker-compose.yaml messageapp
Creating network messageapp_frontend
Creating network messageapp_backend
Creating service messageapp_app
Creating service messageapp_mongo
Creating service messageapp_lbapp
```

* Increase the number of **app** service instances. Since we used `stack deploy` all the services in our application are prefaced with the name of the stack, in this case `messageapp`. So to scale the app service enter this command:

```bash
$ docker service scale messageapp_app=3

messageapp_app scaled to 3
overall progress: 3 out of 3 tasks
1/3: running
2/3: running
3/3: running
verify: Service converged
```

Our application is still available through http://localhost:8000/message

```bash
# find the container ID for the lb-dns
$ docker container ls
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS               NAMES
a122ec48cc14        jimmyarms/lb-dns:001        "nginx -g 'daemon of…"   6 minutes ago       Up 6 minutes        80/tcp, 443/tcp     messageapp_lbapp.1.odl7kfllftqsibtsf4q880rlt
f6f94f168906        jimmyarms/labs-nodejs:001   "docker-entrypoint.s…"   6 minutes ago       Up 6 minutes        80/tcp              messageapp_app.1.nwiqkf5cv0ycnc30thae09z4j
f62b45ccdbb3        mongo:4.2                   "docker-entrypoint.s…"   6 minutes ago       Up 6 minutes        27017/tcp           messageapp_mongo.1.regybe6mzoke5g508n7wct5q7

# exec in to the lb-dns container
$ docker exec -it a122 /bin/sh

# verify you can ping the app service
> ping apps -c 3
PING apps (10.0.9.2): 56 data bytes
64 bytes from 10.0.9.2: icmp_seq=0 ttl=64 time=0.093 ms
64 bytes from 10.0.9.2: icmp_seq=1 ttl=64 time=0.123 ms
64 bytes from 10.0.9.2: icmp_seq=2 ttl=64 time=0.175 ms
--- apps ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.093/0.130/0.175/0.034 ms

# now try mongo
> ping mongo -c 3
ping: unknown host
```

---

## Clean-up

Congratulations! You've now used the Docker Swarm orchestrator! The same techniques in this exercise will work on a single node like Docker Desktop, or across an entire cluster.

In the final section we'll transform our Compose-based services in to a Docker App, and deploy to Kubernetes
