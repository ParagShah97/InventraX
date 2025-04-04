# Docker

Here I am containerizing the project with the help of Docker. That is we will bundle all the (libs, config, Apps, and OS) and any new user can use the bundle (with all the required versions) and run the application.

## Docker Basics

- **Docker Image**: Contains (libs, config, Apps, and OS) and is stored in the Docker registry.
- **Container**: A running instance of a Docker image.
- **Layers**: Each configuration, library, etc., is a separate layer with a unique ID.

## Commands

### Pull and Run Images
```sh
docker pull <name>
docker pull <name:version>  # Example: docker pull redis:6.7.2
```

### Listing and Managing Images
```sh
docker images  # List all images
docker rmi <IMAGE_ID/NAME>  # Remove an image
```

### Running Containers
```sh
docker run --name <CONTAINER_NAME> <IMAGE_NAME:latest/IMAGE_ID>
docker run --name <CONTAINER_NAME> -p <HOST_PORT:CONT_PORT> -d <IMAGE_NAME:latest/IMAGE_ID>
```

### Managing Containers
```sh
docker ps  # List running containers
docker ps -a  # List all containers
docker stop <CONTAINER_ID/NAME>  # Stop a container
docker rm <CONTAINER_ID/NAME>  # Remove a container
docker logs <CONTAINER_ID>  # View logs of a container
docker exec -it <CONTAINER_ID/NAME> /bin/sh  # Access container shell
```

### System Cleanup
```sh
docker system prune -a  # Remove all unused containers, images, and networks
```

## Inventrax Containerization

### Service Registry

1. Build the JAR file:
```sh
mvnw clean install
```
2. Create a `Dockerfile` in the service registry folder:
```dockerfile
FROM openjdk:21
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} serviceregistry.jar
EXPOSE 8761
ENTRYPOINT ["java", "-jar", "/serviceregistry.jar"]
```
3. Build and run the Docker image:
```sh
docker build -t paragshah07/serviceregistry:0.0.1 .
docker run --name serviceregistry -p 8761:8761 -d <IMAGE_ID>
```

### Config Server & Cloud Gateway

Follow similar steps as the service registry.
Run with environment variables:
```sh
docker run --name configserver -p 9296:9296 -e EUREKA_SERVER_ADDRESS=http://host.docker.internal:8761/eureka -d <IMAGE_ID>
docker run --name cloudgateway -p 9090:9090 -e EUREKA_SERVER_ADDRESS=http://host.docker.internal:8761/eureka -e CONFIG_SERVER_URL=host.docker.internal -d <IMAGE_ID>
```

### Order, Product & Payment Services

```sh
docker run --name orderservice -p 8082:8082 -e EUREKA_SERVER_ADDRESS=http://host.docker.internal:8761/eureka -e CONFIG_SERVER_URL=host.docker.internal -e DB_HOST=host.docker.internal -d <IMAGE_ID>
docker run --name productservice -p 8080:8080 -e EUREKA_SERVER_ADDRESS=http://host.docker.internal:8761/eureka -e CONFIG_SERVER_URL=host.docker.internal -e DB_HOST=host.docker.internal -d <IMAGE_ID>
docker run --name paymentservice -p 8081:8081 -e EUREKA_SERVER_ADDRESS=http://host.docker.internal:8761/eureka -e CONFIG_SERVER_URL=host.docker.internal -e DB_HOST=host.docker.internal -d <IMAGE_ID>
```

## Docker Compose

### Creating a `docker-compose.yml` file
```yaml
version: '3'
services:
  serviceregistry:
    image: 'paragshah07/serviceregistry:0.0.1'
    container_name: serviceregistry
    ports:
      - '8761:8761'
  configserver:
    image: 'paragshah07/configserver:0.0.1'
    container_name: configserver
    ports:
      - '9296:9296'
    environment:
      - EUREKA_SERVER_ADDRESS=http://host.docker.internal:8761/eureka
    depends_on:
      - serviceregistry
  cloudgateway:
    image: 'paragshah07/cloudgateway:0.0.1'
    container_name: cloudgateway
    ports:
      - '9090:9090'
    environment:
      - EUREKA_SERVER_ADDRESS=http://host.docker.internal:8761/eureka
      - CONFIG_SERVER_URL=host.docker.internal
    depends_on:
      - configserver
```

### Running and Cleaning Up Containers
```sh
docker-compose -f docker-compose.yml up -d  # Start services
docker-compose -f docker-compose.yml down  # Remove all services
```

### Docker Compose with Health Check
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://configserver:9296/actuator/health"]
  interval: 10s
  timeout: 5s
  retries: 5
```

This README provides the necessary steps for containerizing and running services efficiently using Docker and Docker Compose.

-------------------------------------------------------------------------------------------------------------------------------

# Kubernetes Overview

Kubernetes (K8s) is an open-source **container orchestration platform** that automates the **deployment**, **scaling**, and **management** of containerized applications. It helps manage clusters of hosts running containers in a highly efficient, resilient, and scalable manner.

---

## 🧱 Kubernetes Architecture

Kubernetes architecture follows a **Master-Worker** model:

```
Master Node(s) <-----> Worker Node(s)
```

---

## ⚙️ Master Node Components

The **Master Node** is the control plane of the cluster and is responsible for managing the state of the cluster.

### 🔗 `kube-apiserver`

- Serves as the **front end** of the Kubernetes control plane.
- Exposes the **Kubernetes API**.
- Designed to **scale horizontally** by running multiple instances and balancing traffic.

### 🧠 `controller-manager`

- Watches the state of the cluster and makes changes to bring the actual state to the desired state.
- Includes:
  - **Node Controller** – manages node status.
  - **Replication Controller** – ensures the correct number of pod replicas.
  - **Endpoints Controller** – manages endpoint objects.
  - **Service Account & Token Controller** – manages access credentials.

### 🧭 `kube-scheduler`

- Assigns newly created Pods to nodes.
- Determines pod placement based on resource requirements, policies, and affinity rules.

### 🗃️ `etcd`

- A **highly-available**, consistent **key-value store** used as the **backing store** for all cluster data.
- Essential for cluster state management.
- Requires a [backup plan](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster) to avoid data loss.

---

## 👷 Worker Node Components

Worker Nodes are where **application containers** are deployed and run.

### 🧑‍🏭 `kubelet`

- An agent running on every worker node.
- Ensures that containers are running in their respective Pods as defined in the PodSpecs.

### 📦 Pods

- The **smallest deployable unit** in Kubernetes.
- Each Pod has a **unique IP address**.
- Can contain **one or more containers**.
- Pods are connected via **Services**, which abstract dynamic Pod IPs.

### 🔌 Services

Enable communication across Pods and external clients:

- **Internal Service**: For communication within the cluster.
- **External Service**: Exposes Pods to the outside world.

Services support **load balancing** across multiple Pods.

### ⚙️ ConfigMaps

- Used to store **non-confidential configuration data**.
- Allows separating configuration from application code.

### 🔐 Secrets

- Used to store **sensitive information** (passwords, tokens, etc.).
- Data is stored in **base64-encoded** format.

### 💾 Volumes

- Provide **persistent storage** for Pods.
- Ensures data survives Pod restarts or failures.

Types of storage:
- **Local Volumes**: Inside the cluster.
- **Remote Volumes**: Outside the cluster (e.g., cloud storage).

### 📊 Deployment

- Describes the **desired state** of Pods and ReplicaSets.
- Ensures the specified number of pod replicas are running at all times.
- Useful for rolling updates, rollbacks, and scaling.

---

## 🔀 `kube-proxy` (Optional Component)

- A **network proxy** that runs on each node.
- Implements part of the Kubernetes **Service** abstraction.
- Handles routing and **load balancing** for network traffic to Pods.

---

## ✅ Summary

| Component        | Purpose                                               |
|------------------|--------------------------------------------------------|
| kube-apiserver   | Frontend to Kubernetes API                            |
| controller-manager | Monitors and maintains desired state of the cluster |
| kube-scheduler   | Assigns Pods to nodes                                 |
| etcd             | Persistent, consistent cluster data store             |
| kubelet          | Ensures containers run as expected on a node          |
| Pods             | Smallest deployable unit with container(s)            |
| Services         | Abstraction for Pod communication & load balancing    |
| ConfigMap        | Stores configuration data                             |
| Secrets          | Stores sensitive information                          |
| Volumes          | Provides persistent storage                           |
| Deployment       | Manages desired state and replicas                    |
| kube-proxy       | Network traffic routing                               |

