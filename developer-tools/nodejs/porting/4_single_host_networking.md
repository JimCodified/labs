# Container networking on a single Docker host

In this exercise, we will bring up both of the containers for our application and connect them to each other so our Node.js API can communicate with our MongoDB database.

## Default networks

By default, your Docker Engine already has some default networks:

```bash
$ docker network ls
NETWORK ID          NAME            DRIVER
d87b8fc4c466        bridge          bridge
efaf610f57a5        host            host
f7d0de539edd        none            null
```

By default (if no `--net` option is provided), Docker Engine will attach each container to the bridge network (ID _d87b8fc4c466_ in the output above, although network IDs will vary).

## Default bridge network

Let's run 2 containers using the default bridge network without using the `--net` option, to see how they behave.

```
$ docker run --name mongo -d mongo:4.2
$ docker run --name box -d busybox top
```

Make sure the containers are listed in the bridge network. You'll need your bridge NETWORK ID from the earlier command and substitute that in place of `d87b8fc4c466` in the example below (or you could just use the name `bridge` instead of the ID):

```bash
$ docker network inspect --format='{{json .Containers}}' d87b8fc4c466 | python -m json.tool
{
    "0b8fedf4613c7275d89861037ea1b23ad4d65ab10f16df67bf976d9cb5652311": {
        "EndpointID": "0cf0cd3b2e0438c6f68c6a1e2f7587b63c48bda74911af55d1040f0d2fb117d2",
        "IPv4Address": "172.17.0.3/16",
        "IPv6Address": "",
        "MacAddress": "02:42:ac:11:00:03",
        "Name": "mongo"
    },
    "6cb5e5f4a1bcc37925407b39f2dde41f2b370fc48a21f8289da91d17b3763a4c": {
        "EndpointID": "2a6412d3c3c25545a59ea148e317b2046965c0fe5c1eeae2c51f4f882aaa6b36",
        "IPv4Address": "172.17.0.2/16",
        "IPv6Address": "",
        "MacAddress": "02:42:ac:11:00:02",
        "Name": "box"
    }
}
```

The `Name` field tells us that both the `mongo` and `box` containers are connected to the bridge network. However, **they cannot address each other by their names** because we did not expose or publish any ports for either container to use. On the default bridge network the default behavior is to isolate every container.

```bash
# start a new busybox container and run an interactive shell:
$ docker run -ti busybox /bin/sh

# try to ping our other containers:
/ $ ping mongo
ping: bad address 'mongo'
/ $ ping box
ping: bad address 'box'

/ $ exit
```

## User defined bridge network

When we create a user defined network, the behaviour is different than the default bridge network.

Let's create a user defined bridge network with Docker network commands

```bash
# cleanup our earlier containers:
$ docker container stop box mongo && docker container rm box mongo
box
mongo

# create a new network
$ docker network create mongonet
ce9ea3b69d6ee2ecf56b40bd35b8a43f8505c8ca0473bc37bdede3711ecf60c1

$ docker network ls
NETWORK ID          NAME            DRIVER
d87b8fc4c466        bridge          bridge
efaf610f57a5        host            host
ce9ea3b69d6e        mongonet        bridge
f7d0de539edd        none            null
```

Let's now run 2 containers in the newly defined network

```
$ docker run --name mongo --net mongonet -d mongo:3.2

$ docker run --net mongonet -ti busybox /bin/sh
/ # ping -c 3 mongo
PING mongo (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.058 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.085 ms
64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.072 ms

--- mongo ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.058/0.071/0.085 ms

/ # exit
$ docker container stop box mongo && docker container rm mongo
box
mongo
````

Containers can be addressed by their name through the DNS name server embedded in Docker 1.10+

## Running our full application

Run db and application containers in the new bridge network

```bash
$ docker run --name mongo --net mongonet -d mongo:4.2
0dd9fd6a3108caa7cac6709ce89e60c5e474b4a051becac7562c4f30efeba293

$ docker run --name app --net mongonet -p 8000:1337 -d -e "MONGO_URL=mongodb://mongo/messageApp" message-app:latest
2d52bb5772f771b8ddb9177cbd5c9a53e6e1915743a129a09add10805b48a967
```

Note: The `-e` switch sets an environment variable MONGO_URL inside the container which directly uses **mongo** container’s name

Test HTTP Rest API

```bash
# Create a  new message
$ curl -XPOST http://localhost:8000/message?text=hello
{
  "text": "hello",
  "createdAt": "1569451938332",
  "updatedAt": "1569451938332",
  "id": "57558221a4461312009ce88c"
}

# Retrieve the list of message and make sure the previous message is present
$ curl -XGET http://localhost:8000/message
[
  {
    "text": "hello",
    "createdAt": "1569451938332",
    "updatedAt": "1569451938332",
    "id": "57558221a4461312009ce88c"
  }
]

# cleanup
$ docker container stop mongo app && docker container rm mongo app
mongo
app
mongo
app
```

The application container (named **app**) is connected to mongo container using container name (named **mongo**).

**Congratulations! You've successfully containerized your application!**

However, there is a better way to specify and run multiple containers together rather than manually creating a network, then launching each container separately with all the switches to wire it up.

## Specifying the application services with Docker Compose

The following file defines the whole application. Better yet, it's using images from Docker Hub so you can take this Compose file to any system with Docker Engine and run it the same way, with the same images.

Copy this code into a file on your system named `docker-compose.yaml` - don't forget to substitute your own Docker Hub image for the `app:` service.

```yaml
version: '3.6'
services:
  mongo:
    image: mongo:4.2
    volumes:
      - mongo-data:/data/db
    expose:
      - "27017"
  app:
    # NOTE: you should substitute your own Docker Hub info below
    image: jimmyarms/labs-nodejs:001
    ports:
      - "1337"
    links:
      - mongo
    depends_on:
      - mongo
    environment:
      - MONGO_URL=mongodb://mongo/messageApp
volumes:
  mongo-data:
```

The important part of this file:

* Definition of 2 services
  * database service: **mongo**
  * application service: **app**
* Link between **app** and **mongo** services done through the MONGO_URL environment variable (using **mongo** service name)
* Port mapping
  * mongo service exposes port 27017 (default MongoDB port) only to the other services listed in this Compose file (not to the Docker host)
  * app service port is mapped to a random port on the host (as no host port as been defined)
* Definition of a user defined volume for mongodb data folder

## Lifecycle and scalability

The following commands are some of the main ones to interact with the application

* Start the application ```docker-compose up -d```  (-d option enables the application to run in background)
* Check the status of each services conposing the application ```docker-compose ps```
* Stop the application ```docker-compose stop```
* Scale the app service changing the number of instances ```docker-compose scale app=3```

![3 api containers](https://dl.dropboxusercontent.com/u/2330187/docker/labs/node/single_host_net_1.png)

Several containers of the app service (our Node.js API) are running and are accessible through random port number of the Docker host. Wow are the new instanciated containers addressed ?

=> Need to add a load balancer that will be updated each time a container is created or removed and that will forward each request to a running instance of the app service.

## Usage of dockercloud/haproxy image

[dockercloud/haproxy](https://store.docker.com/images/haproxy) is a good candidate to be used in front of our **app** service.
It will update it's configuration each time a container is started / stopped.

![load balancer](https://dl.dropboxusercontent.com/u/2330187/docker/labs/node/single_host_net_2.png)

## Adding load balancer to our Compose file

The new version of our docker-compose.yml is

```
version: '3'
services:
  mongo:
    image: mongo:3.2
    volumes:
      - mongo-data:/data/db
    expose:
      - "27017"
 lbapp:
    image: dockercloud/haproxy
    links:
      - app
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "8000:80"
  app:
    image: message-app
    expose:
      - "1337"
    links:
      - mongo
    depends_on:
      - mongo
    environment:
      - MONGO_URL=mongodb://mongo/messageApp
volumes:
  mongo-data:
```

The load balancer service has been added to the picture.
Each request coming to port 8000 of the host (mapped with port 80 of lbapi) will go to the api through the load balancer.

## Test our application

Run the new version of our compose file and specify the number of instances of the **app** service
* ```docker-compose up```
* ```docker-compose scale app=3```

Let's just test the creation and retrieval of a message

```
$ curl -XPOST http://192.168.99.100:8000/message?text=hola
{
  "text": "hola",
  "createdAt": "2016-06-08T13:30:18.298Z",
  "updatedAt": "2016-06-08T13:30:18.298Z",
  "id": "57581deacde05a1200877fa2"
}
$ curl -XGET http://192.168.99.100:8000/message
[
  {
    "text": "hola",
    "createdAt": "2016-06-08T13:30:18.298Z",
    "updatedAt": "2016-06-08T13:30:18.298Z",
    "id": "57581deacde05a1200877fa2"
  }
]
```

Seems to be good :)
