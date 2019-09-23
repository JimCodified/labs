# What we did in the _Node.js Programming in Docker Containers_ tutorial

* Setup Docker for a simple Node.js / MongoDB application

* Created a container image for the application
  * containing all the parts to run the application (runtime Node.js, libraries, application code)

* Pushed the container image to Docker Hub to make it available elsewhere

* Scaled the service from a single node, to a cluster, to Docker Swarm and Kubernetes
  * on a single node (for dev / test purposes)
  * on a cluster of Docker hosts
  * on a Docker Swarm
  * on Kubernetes

* We also looked at several Docker components and how they are integrated together
