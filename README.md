# multi-tier-cloud
Docs will go here

## Getting Started
WORK IN PROGRESS

### 1. Bootstrap
https://github.com/Multi-Tier-Cloud/bootstrap-node

TODO: Need to make configurable

For now, rely on hardcoded bootstrap nodes running in SAVI

### 2. Allocator
https://github.com/Multi-Tier-Cloud/service-manager

```
$ cd service-manager/allocator
$ go build
$ sudo ./allocator
```

Run an allocator on each machine that you want to spawn new microservice instances.
Needs to be run with sudo so it can issue Docker commands.
Requires Docker to already be installed on the machine (TODO: make it use only client library instead of system commands).

### 3. Monitoring
https://github.com/Multi-Tier-Cloud/monitoring

TODO: System currently hardcoded to request data from existing Prometheus instance. Need to make this configurable. This section will explain how the existing system was set up.

```
$ cd monitoring/ping-monitor
$ go build
```

Run ping-monitor somewhere. It will ping all nodes in the network and export these metrics for Prometheus to scrape.

Create a prometheus.yml with a scrape_configs, static_configs targets set to \<IP of machine running ping-monitor>:8888

```
...

scrape_configs:

    ...

    static_configs:
        - targets: ['10.11.17.1:8888']
```

Run prometheus.
```
./prometheus --config.file=prometheus.yml --web.listen-address=0.0.0.0:7777
```

### 4. Hash Lookup
https://github.com/Multi-Tier-Cloud/hash-lookup

```
$ cd hash-lookup/hl-service
$ go build
$ ./hl-service --new-etcd-cluster --etcd-ip <machine's IP address>
```

Run at least 1 hash-lookup service somewhere in the network. Will be used later to register a new microservice with the system.

### 5. Example "Hello world" microservice
https://github.com/Multi-Tier-Cloud/demos

```
$ cd demos/helloworld/helloworldserver
$ go build
```

Server that responds to requests with "Hello, world". The next few steps will go over how to automatically deploy to the network. For now, you can test it out by runnning `./helloworldserver <port>` where `<port>` is the port the server will listen on. `curl 127.0.0.1:<port>` should return "Hello, world".

### 6. Containerize microservice and add to hash-lookup
TODO: automate this process. For now, follow these manual steps.

https://github.com/Multi-Tier-Cloud/service-manager/tree/master/proxy

```
$ cd service-manager/proxy
$ go build
```

Proxy is needed for a microservice to communicate with rest of system/network. A proxy needs to be packaged with each microservice.


Create a Docker image for the microservice. You will also need a DOckerHub account to host the image. Start by making a new directory. From demos/helloworld/helloworldserver, copy in the helloworldserver executable. From service-manager/proxy, copy in the proxy executable and perf.conf. Create a Dockerfile as shown below.

```Dockerfile
FROM ubuntu:16.04

WORKDIR /app

COPY helloworldserver .
COPY proxy .
COPY perf.conf .

ENV PROXY_PORT=4201

ENV PROXY_IP=127.0.0.1
ENV SERVICE_PORT=8080

CMD ./proxy $PROXY_PORT hello-world-server $PROXY_IP:$SERVICE_PORT & ./helloworldserver $SERVICE_PORT
```

Note the special environment variables PROXY_PORT, PROXY_IP, and SERVICE_PORT. These variables will be set dynamically when a new container is spun up (so the values listed in the Dockerfile don't mean anything). PROXY_PORT is the port that the proxy should listen on. PROXY_IP is the IP address of the machine that the proxy/microservice will be running on. SERVICE_PORT is the port that the microserivce should listen on.


On your DockerHub, create a repo named hello-world-server. Build the image. Make sure to use your DockerHub username so you can push to DockerHub.

```
$ sudo docker image build -t <DockerHub username>/hello-world-server:1.0 .
```

Push it.

```
$ sudo docker image push <DockerHub username>/hello-world-server:1.0
```
After pushing, Docker will report the image's "digest", which is a cryptographic hash of the image. Record this value somewhere. If you forget, don't worry, you can always find it on DockerHub.

Next, to register our microservice with the system we'll need to save a copy of the Docker image.

```
$ sudo docker save -o hello-world-server.1.0.tar <DockerHub username>/hello-world-server:1.0
```

We will use the hash lookup cli. 

```
$ cd hash-lookup/hl-cli
$ go build
$ ./hl-cli add --file hello-world-server.1.0.tar <DockerHub username>/hello-world-server@sha256:<image digest/hash> hello-world-server
```

This will add the microservice to the hash lookup service, effectively making it deployable to the system. `<image digest/hash>` is the image digest mentioned above. We are giving it the name "hello-world-server".

### 7. Test it out

We can make a request for the "hello-world-server" and the system will automatically allocate a new instance of it on one of the machines that has an allocator running.

We will need our own proxy to communicate with the system.

```
$ ./proxy 4200
```

Ask the proxy to send a request to an instance of "hello-world-server".

```
$ curl 127.0.0.1:4200/hello-world-server
```

You should receive a response of "Hello, world". Check that on one of the machines running an allocator, when you run `sudo docker ps` you see a container running the hello-world-server.
