# Node.js Programming in Docker Containers

# Purpose

This tutorial starts with a simple Node.js application (HTTP Rest API built with [Sails.js](http://sailsjs.org/)) and details the steps needed to containerize it and ensure its scalability.

The application stores data in a MongoDB database. This tutorial does not address the scaling of the MongoDB part.

Note: Do not hesitate to provide any comments / feedback you may have, that will help make this tutorial better.

# Prerequisites

Some of the Docker basics will be reviewed but it is recommended to follow [Docker for Beginners](https://github.com/docker/labs/tree/master/beginner) prior to follow this tutorial in order to get a clear understanding of what is inside Docker and how to use it.

# Let's start

[Setup our sample node application](1_node_application.md)

[Create the application's image](2_application_image.md)

[Publish image on Docker Hub](3_publish_image.md)

[Single Docker host networking](4_single_host_networking.md)

[Multiple Docker hosts networking](5_multiple_hosts_networking.md)

[Deploy on a Docker Swarm](6_deploy_on_swarm.md)

[Deploy on Kubernetes](7_deploy_on_kubernetes.md)

# Summary

When you finish you will have covered several important aspects of Docker and hopefully have a better understanding of how you can use Docker in your everyday coding practice.

[What we covered in these exercises](summary.md)

Once again, feedback / comments are more than welcome :)
