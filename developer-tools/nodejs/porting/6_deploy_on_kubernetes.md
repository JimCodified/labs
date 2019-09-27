# Docker Apps and Deploying to Kubernetes

## Our story so far

So far, you have accomplished the following:

* Took a Node.js application that uses the SailsJS framework and containerized it
* Added a database service and connected the app service to the database
* Added some basic orchestration on Docker Swarm

In this exercise we will add two more accomplishments:

1. Convert our Compose-based services in to a _Docker App_
2. Deploy our app on Kubernetes

---

## Prerequisites

You'll obviously need to have Kubernetes installed and running at this point. _Docker App_ is also an additional component that might not be installed already.

### Docker Desktop

If you're using Docker Desktop you have everything you need. You just need to enable it. Check [the setup instructions](0_setup.md) to see how to enable Kubernetes. Also make sure you have **Experimental Features** enabled (`Preferences --> Daemon --> Experimental Features should be checked`)

### Standalone Docker Engine

If you're using a standalone Docker Engine you will need to install Kubernetes and Docker App on your own:

* [Minikube](https://github.com/kubernetes/minikube/releases) is available
* [Docker App](https://github.com/docker/app/releases/) is freely available as well (v0.8.0 at the time of writing)
* [Compose on Kubernetes](https://github.com/docker/compose-on-kubernetes/releases) will also be required. We will not be covering how to write Kubernetes YAML in this exercise!

---

## Quick Kubernetes Deployment

We don't have to convert to a Docker App to run on Kubernetes. In fact, you can use almost the exact same command you used in the previous exercise and deploy to Kubernetes right now, if you want:

```bash
$ docker stack deploy -c ./docker-compose.yaml --orchestrator kubernetes messageapp

top-level network "backend" is ignored
top-level network "frontend" is ignored
service "mongo": network "backend" is ignored
service "lbapp": network "frontend" is ignored
service "app": network "backend" is ignored
service "app": network "frontend" is ignored
service "app": depends_on are ignored
Waiting for the stack to be stable and running...
app: Ready		[pod status: 1/1 ready, 0/1 pending, 0/1 failed]
lbapp: Ready		[pod status: 1/1 ready, 0/1 pending, 0/1 failed]
mongo: Ready		[pod status: 1/1 ready, 0/1 pending, 0/1 failed]

Stack messageapp is stable and running
```

That's all it takes. You can run `kubectl` commands (try `kubectl get pods`) to verify that everything is running. The _Compose on Kubernetes_ controller converted our Docker Compose format to Kubernetes yaml and started all the pods for us.

Unfortunately, as you can see, the network options were all ignored so our app won't function right now. Clean it up and we'll fix things:

```
$ docker stack rm messageapp
```

## Creating a Docker App

# TBC...
