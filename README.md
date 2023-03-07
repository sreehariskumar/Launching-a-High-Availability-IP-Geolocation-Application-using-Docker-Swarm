# Launching a High Availability IP Geolocation Application using Docker Swarm

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

![swarm drawio](https://user-images.githubusercontent.com/68052722/223336304-25b54a2a-3bd6-44d6-b55a-0e1fad89853f.png)


By utilizing the benefits of containerization, I have built an IP geolocation solution that can easily be deployed and managed, even in large and complex environments. With a focus on high availability, our application offers an unparalleled level of reliability, scalability, and performance. Let’s explore the journey of developing this application, and how Docker Swarm has helped me achieve this goal.

### Introduction to Docker Swarm
[Docker Swarm](https://docs.docker.com/engine/swarm/) is a native clustering and orchestration solution for Docker containers. It allows you to manage a group of Docker nodes as a single virtual system, making it easy to deploy and manage applications at scale. With Docker Swarm, you can create and manage a swarm of Docker nodes, and then deploy services to the swarm. Docker Swarm automatically distributes the services across the nodes in the swarm, ensuring that the services remain available even if one or more nodes fail. It also provides tools for scaling the services up or down as needed and rolling out updates to the services without downtime. Overall, Docker Swarm provides a powerful and flexible solution for managing and deploying applications in a production environment.
You need not have to install docker swarm separately as **Docker Swarm mode is built into the Docker Engine.**

In Docker Swarm, there are two types of nodes:
a) **Manager nodes** are responsible for controlling and coordinating the swarm. They handle tasks such as accepting service definitions, scheduling tasks, and monitoring the state of the swarm. In a multi-node swarm, there must be at least one manager node and there can be multiple manager nodes for redundancy and high availability.

b) **Worker nodes** are the nodes that run the tasks and services defined in the swarm. They do not have the ability to control the swarm, and their primary function is to execute the tasks assigned to them by the manager nodes.

Having a combination of manager and worker nodes allows you to create a highly available and scalable cluster, where the manager nodes handle the orchestration of the services and the worker nodes execute the tasks. If a worker node fails, the manager nodes will automatically reschedule the tasks on another available worker node, ensuring that the services remain running and available.


| Docker | Version |
| ------ | ------ |
| Client | 20.10.17 |
| Server | 20.10.17 |

### Requirements

- Launch 4 instances (1 manager and 3 workers).
- Docker must be [installed](https://www.docker.com/) and docker daemon running on all instances.
- The user running the docker commands may be added to the docker group or else you may need sudo privilege to run the commands. (optional)
- Create an account in [IP Geolocation API](https://ipgeolocation.io/) & copy the API key.
- A valid domain name.
- An ACM [certificate](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html). 

### Manager Node
- Log in to the manager instance and make sure docker is installed. Change the hostname of the instance to “manager” for easy identification.
- Run the following command to initialize the docker swarm:

```s
[ec2-user@manager ~]$ docker swarm init
```
```s
Swarm initialized: current node (rhrzijigbe35akwsuqzced7ow) is now a manager.
To add a worker to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-3ye7nynchxc4kzcpwwhq1yk9vbvpvhbpd5qrea9nqayq0shryo-es9i2krf5fjirq2up6qf7h5ls 172.31.8.122:2377
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
- You will receive a join token which should be executed on worked nodes for them to be added to the cluster.

### Worker Node
- Log in to each worker node and ensure that docker is installed. Change the hostname of the worker instances to worker1, worker2 & worker3 respectively.
- Run the following command on the worker nodes to add them as worker nodes:

```s
docker swarm join --token SWMTKN-1-3ye7nynchxc4kzcpwwhq1yk9vbvpvhbpd5qrea9nqayq0shryo-es9i2krf5fjirq2up6qf7h5ls 172.31.8.122:2377
```

Once completed, go back to the manager node and run the following command to check whether the worker nodes have been added correctly:

```s
[ec2-user@manager ~]$ docker node ls
```
```s
ID                            HOSTNAME                                   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
rhrzijigbe35akwsuqzced7ow *   manager.ap-south-1.compute.amazonaws.com   Ready     Active         Leader           20.10.17
51q3epwjoezb2fx24eg7oiskj     worker1.ap-south-1.compute.amazonaws.com   Ready     Active                          20.10.17
0w4a9i725w3wln0e9qha6vvga     worker2.ap-south-1.compute.amazonaws.com   Ready     Active                          20.10.17
pdnoxuq3tha87r6cnx4cvrunr     worker3.ap-south-1.compute.amazonaws.com   Ready     Active                          20.10.17
```

Based on the node’s availability status, containers are constructed. On active nodes, new containers will be constructed. This implies that the manager node will also be used to build the containers. To prevent this, the management node’s status must be changed to [Drain](https://docs.docker.com/engine/swarm/swarm-tutorial/drain-node/), which stops it from receiving new tasks.


### Changing the Availability Status of the Manager Node to Drain

```s
docker node update --availability drain manager.ap-south-1.compute.amazonaws.com
```
```s
docker node ls
```
```s
ID                            HOSTNAME                                   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
rhrzijigbe35akwsuqzced7ow *   manager.ap-south-1.compute.amazonaws.com   Ready     Drain          Leader           20.10.17
51q3epwjoezb2fx24eg7oiskj     worker1.ap-south-1.compute.amazonaws.com   Ready     Active                          20.10.17
0w4a9i725w3wln0e9qha6vvga     worker2.ap-south-1.compute.amazonaws.com   Ready     Active                          20.10.17
pdnoxuq3tha87r6cnx4cvrunr     worker3.ap-south-1.compute.amazonaws.com   Ready     Active                          20.10.17
```
As you can see that the availability status of the manager node has been changed to **“Drain”**.

### Adding Node Labels
- In this project, we will be running the api containers and the frontend containers in worker1 and worker2 only.
- The worker3 is solely used for running a redis container for caching.
- We can use labels to ensure that certain containers are only placed on worker nodes that meet specific requirements. We can use labels to ensure that certain containers are only placed on worker nodes that meet specific requirements.

We will be using the **“service=api-service”** label for worker1 & worker2. And the **“service=redis-service”** label for worker3.

Remember: **docker node update** only accepts exactly 1 argument. So, each label has to be added individually.
```s
docker node update --label-add service=api-service worker1.ap-south-1.compute.amazonaws.com

```
```s
docker node update --label-add service=api-service worker2.ap-south-1.compute.amazonaws.com
```
```s
docker node update --label-add service=redis-service worker3.ap-south-1.compute.amazonaws.com
```

To check for the label of a node.

```s
docker node inspect worker1.ap-south-1.compute.amazonaws.com

"Labels": {
    "service": "api-service"
},
```
Similarly, check the label has been added for all the worker nodes.

### Creating an Overlay Network
An overlay network in Docker Swarm is a virtual network that enables containers running on different worker nodes to communicate with each other as if they were all connected to the same network. An overlay network is created and managed by Docker Swarm and provides a way to securely connect containers in the swarm, regardless of the worker nodes they are running on.

The purpose of an overlay network in Docker Swarm is to provide a networking solution for applications running in the swarm. Without an overlay network, containers running on different worker nodes would not be able to communicate with each other, making it difficult to build and run distributed applications. By creating an overlay network, you can easily connect containers running on different worker nodes, allowing them to communicate and work together to provide a seamless experience for your users.

```s
docker network create --driver overlay geo-net
```
```s
docker network ls

NETWORK ID     NAME              DRIVER    SCOPE
0ff8bddf838c   bridge            bridge    local
c5ae20a136fc   docker_gwbridge   bridge    local
lfu0cl07jgx2   geo-net           overlay   swarm     <==
b7e2aefe7503   host              host      local
dmb9ti5i1frs   ingress           overlay   swarm
411f646cc1d8   none              null      local
```

### Creating Redis Service
Redis container is used as a caching service for our application.

Here, **only 1** redis container is created in a single instance.
Imagine if multiple redis containers were being used for caching and, an api container receives a request, it first connects to the redis container to see if the requested data is present there. If not, the api container then contacts with the api service provider, and the requested data is then cached in the redis container and served.

If a new request for the same IP address is received on the api container and this time the request is being forwarded to another redis container it may not have the cached data. Here, the api container repeats the process, and this time, caching is done for the same IP address on the new redis container. In order to prevent this, only one redis container is utilised.


```s
docker service create --name geo-redis --constraint node.labels.service==redis-service --network geo-net redis:latest

y226j4sulmlcbngghatcbpigi
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 
```

#### Explanation:
- This command creates a Docker service called “geo-redis” using the latest version of the Redis image.
- **--constraints** are condition which specifies that the service will only run on nodes with the label **“service”** set to **“redis-service”**, which is worker3.
- The service is also connected to the **“geo-net”** network.

To list the created service:
```s
docker service ls

ID             NAME        MODE         REPLICAS   IMAGE          PORTS
y226j4sulmlc   geo-redis   replicated   1/1        redis:latest
```

To list the nodes that are associated with the service:

```s
docker service ps geo-redis

ID             NAME          IMAGE          NODE                                       DESIRED STATE   CURRENT STATE            ERROR     PORTS
9tpotiex5e49   geo-redis.1   redis:latest   worker3.ap-south-1.compute.amazonaws.com   Running         Running 47 seconds ago
```

### Creating API Services

```s
docker service create --name geo-api -p 8081:8080 --constraint node.labels.service==api-service --replicas 4 --network geo-net -e REDIS_PORT="6379" -e REDIS_HOST="geo-redis" -e APP_PORT="8080" -e API_KEY_FROM_SECRETSMANAGER="True" -e SECRET_NAME="xxxxx" -e SECRET_KEY="xxxxx" -e REGION_NAME="xxxxx" sreehariskumar/ip-geo-location-finder
```

#### Explanation:
- This command creates a Docker service called **“geo-api”** using the image **“sreehariskumar/ip-geo-location-finder:latest”**.
- The service maps the host’s port **8081** to the container’s port 8080.
- The service will only run on nodes with the label **“service”** set to **“api-service”**.
- The number of **replicas** of this service is 4.
- The service is connected to the **“geo-net”** network.

 The following environment variables are passed to the service:
 - **REDIS_PORT** is set to “6379”
 - **REDIS_HOST** is set to “geo-redis”.
 Here, the name of the redis service is mentioned for the api service to communicate with the redis service.
 - **APP_PORT** is set to “8080”.
 - **API_KEY_FROM_SECRETSMANAGER** is set to “True”, which defines the API key that should be fetched from our stored secret.
 - **SECRET_NAME** defines the name of our secret.
 - **SECRET_KEY** is key to identifying our secret.
 - **REGION_NAME** is the region of our secret.
 
Listing the nodes that are associate with the service:
```s
docker service ps geo-api

ID             NAME        IMAGE                                          NODE                                       DESIRED STATE   CURRENT STATE           ERROR     PORTS
xagt0bapjjzh   geo-api.1   sreehariskumar/ip-geo-location-finder:latest   worker2.ap-south-1.compute.amazonaws.com   Running         Running 3 minutes ago             
z5xhhgen194o   geo-api.2   sreehariskumar/ip-geo-location-finder:latest   worker1.ap-south-1.compute.amazonaws.com   Running         Running 3 minutes ago             
0xgna3wf4jll   geo-api.3   sreehariskumar/ip-geo-location-finder:latest   worker2.ap-south-1.compute.amazonaws.com   Running         Running 3 minutes ago             
zfmf2n49iocm   geo-api.4   sreehariskumar/ip-geo-location-finder:latest   worker1.ap-south-1.compute.amazonaws.com   Running         Running 3 minutes ago
```
As you can see the replicas have been created across worker1 and worker2.

### Creating Frontend Service
Fronend is just a container that fetches the details of the provided IP address from the api containers and displays it in a readable HTML table format.

```s
docker service create --name geo-frontend -p 8082:8080 --constraint node.labels.service==api-service --replicas 4 --network geo-net -e API_SERVER="geo-api" -e API_SERVER_PORT="8080" -e APP_PORT="8080" sreehariskumar/ipgeolocation-frontend:latest
```

#### Explanation:
- This command creates a Docker service called **“geo-frontend”** using the image **“sreehariskumar/ipgeolocation-frontend:latest”**.
- The service maps the host’s port **8082** to the container’s port 8080.
- The service will only run on nodes with the label **“service”** set to **“api-service”**.
- The number of **replicas** of this service is 4.
- The service is connected to the **“geo-net”** network.

The following environment variables are passed to the service:

- **API_SERVER** is set to “geo-api”.
Here, the name of the api service is mentioned for the frontend service to communicate with the api service.
- **API_SERVER_PORT** is set to “8080”, which defines the api containers are running on port 8080.
- **APP_PORT** is set to “8080”, which specifies the port of the frontend app.

Listing the nodes that are associate with the service:
```s
docker service ps geo-frontend

ID             NAME             IMAGE                                       NODE                                       DESIRED STATE   CURRENT STATE                ERROR     PORTS
xhzkx8j1t4za   geo-frontend.1   sreehariskumar/ipgeolocation-frontend:latest   worker2.ap-south-1.compute.amazonaws.com   Running         Running about a minute ago             
ctcdzz8w7ivp   geo-frontend.2   sreehariskumar/ipgeolocation-frontend:latest   worker1.ap-south-1.compute.amazonaws.com   Running         Running about a minute ago             
g7s77stsdkvq   geo-frontend.3   sreehariskumar/ipgeolocation-frontend:latest   worker2.ap-south-1.compute.amazonaws.com   Running         Running about a minute ago             
var3d0srtsci   geo-frontend.4   sreehariskumar/ipgeolocation-frontend:latest   worker1.ap-south-1.compute.amazonaws.com   Running         Running about a minute ago       
```
As you can see the replicas have been created across worker1 and worker2.

### Load Balancing the Instances
An **nginx** container or an **Application Load Balancer** can be used to balance the traffic here. We’ve used an nginx container previously in an article. This time, we will be using an Application Load Balancer.

### Redirecting HTTP to HTTPS
We will configure the HTTP requests received on the load balancer to redirect to HTTPS.
For this, we need to add a new listener port on the load balancer.

### Creating Records in Route53
We need to create two records in the hosted zone to publicly access the api targets and the frontend targets.


**I hope this project was helpful to you.
Please reach out to me in case of any queries also please share your reviews.**
