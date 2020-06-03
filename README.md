# multi-tier-cloud
Docs will go here

## Getting Started
**WORK IN PROGRESS**

### 1. Bootstrapping the P2P Network
https://github.com/Multi-Tier-Cloud/monitoring

To bootstrap the network, we use the `ping-monitor` tool from the monitoring repo. It listens on a well-known TCP port (default: **TCP port 4001**) and accepts connections from peers.

For each peer that connects, `ping-monitor` will continuously ping it through the overlay (P2P) connection and measure its RTT. The RTT information is available to be scraped by Prometheus at **TCP port 9100**.

To build `ping-monitor`:
```
$ cd monitoring/ping-monitor
$ go build
$ ./ping-monitor [flags ...]
```

Notable flags:
- `-bootstrap`
If a network already exists and the goal is to deploy redundant (i.e. backup) bootstraps, you can run `ping-monitor` with the `-bootstrap` flag and specify the overlay P2P address of a pre-existing bootstrap node.
- `psk`
You can create a _Private Network_ where only nodes with a certain pre-shared key (a passphrase you pass on the CLI) can join. You can specify the passphrase when you execute `ping-monitor` using the `psk` flag.

To list the full set of options: `$ ./ping-monitor -h`

### 2. Allocator
https://github.com/Multi-Tier-Cloud/service-manager

A single instance of the `allocator` should run on each machine that you want to spawn new microservice instances. Currently, the microservice instances are realized using Docker containers (in other words, the Docker engine is a pre-requisite that must be installed).

**Note:** Make sure your account is a member of the `docker` Linux group.
To add your account, you can run: `sudo usermod -aG docker $USER`

To build `allocator`:
```
$ cd service-manager/allocator
$ go build
$ ./allocator -bootstrap <bootstrap-p2p-addr> [flags ...]
```

To run the allocator, you must provide the P2P address of at least one bootstrap node using the `-bootstrap` flag (the flag can be specified multiple times to specify multiple bootstraps).

To list the full set of options: `$ ./allocator -h`

### 3. Hash Lookup
https://github.com/Multi-Tier-Cloud/hash-lookup

The hash lookup service, `hl-service`, is used to translate service hashes into Docker hashes. The Docker hashes are used by instances of `allocator` to pull the Docker image corresponding to a given service, and run it. Currently, the service uses `etcd` as the datastore, and thus it is a pre-requisite that must be installed.

To build `hl-service`:
```
$ cd hash-lookup/hl-service
$ go build
$ ./hl-service --new-etcd-cluster --etcd-ip <machine's IP address> -bootstrap <bootstrap-p2p-addr>
```

When running `hl-service`, you must provide:
1. The IP address of a running instance of `etcd`
2. The P2P address of at least one bootstrap node using the `-bootstrap` flag (the flag can be specified multiple times to specify multiple bootstraps).

To list the full set of options: `$ ./hl-service -h`

Run at least 1 instance of `hl-service` somewhere in the network. Will be used later to register a new microservice with the system.

### 4. Set Up Prometheus
**TODO:** System currently hardcoded to request data from existing Prometheus instance. Need to make this configurable. This section will explain how the existing system was set up.

Download Prometheus: Find the release corresponding to your system at https://prometheus.io/download/ and untar the files.

In the new directory created when you untar'd the files, create a file called `prometheus.yml`. Populate it with the basic setup shown in: https://prometheus.io/docs/prometheus/latest/getting_started/

To configure Prometheus to scrape the `ping-monitor`, add a new job with the `static_configs` targets set to `<IP of machine running ping-monitor>:9100`
- **Note:** Port 9100 is simply the default port that `ping-monitor` listens on for requests from Prometheus. If you changed the default port using its `-prom-listen-addr` flag, then make sure the port you specify here matches.

Example:
```
...

scrape_configs:

    ...
  - job_name: 'ping-monitor'
    static_configs:
        - targets: ['10.11.17.1:9100']
```

You can also specify a new jobs to scrape the `allocator`s (which are currently hard-coded to listen on port 9101), and the `hl-service` (default port 9102)
- **TODO:** Make this port configurable in the `allocator` via flag

Run prometheus.
```
./prometheus
```

You can visit Prometheus' default web portal on port 9090.


### 5. Example "Hello world" microservice
https://github.com/Multi-Tier-Cloud/demos

```
$ cd demos/helloworld/helloworldserver
$ go build
```

Server that responds to requests with "Hello, world". The next few steps will go over how to automatically deploy to the network. For now, you can test it out by runnning `./helloworldserver <port>` where `<port>` is the port the server will listen on. `curl 127.0.0.1:<port>` should return "Hello, world".

### 6. Containerize microservice and add to hash-lookup
**TODO:** automate this process. For now, follow these manual steps.

https://github.com/Multi-Tier-Cloud/service-manager/tree/master/proxy

Containerized services are packaged alongside an instance of `proxy` that runs within the container's network namespace.
The `proxy` is needed for a microservice to communicate with rest of system/network.
To build `proxy`:
```
$ cd service-manager/proxy
$ go build
```

Create a Docker image for the microservice. You will also need a DockerHub account to host the image. Start by making a new directory. From `demos/helloworld/helloworldserver`, copy in the `helloworldserver` executable. From `service-manager/proxy`, copy in the `proxy` executable and `perf.conf`. Create a Dockerfile as shown below.
```Dockerfile
FROM ubuntu:16.04

WORKDIR /app

COPY helloworldserver .
COPY proxy .
COPY perf.conf .

ENV PROXY_PORT=4201

ENV PROXY_IP=127.0.0.1
ENV SERVICE_PORT=8080

CMD ./proxy -bootstrap <bootstrap-p2p-addr> -configfile perf.conf $PROXY_PORT hello-world-server $PROXY_IP:$SERVICE_PORT & ./helloworldserver $SERVICE_PORT
```
**TODO:** Pass the bootstrap options (and potential PSK options) to proxy via environment variables?

Replace `<bootstrap-p2p-addr>` above with the multiaddress of an actual bootstrap node.
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

We will need our own `proxy` to communicate with the system.
```
$ ./proxy -bootstrap <bootstrap-p2p-addr> 4200
```

Ask the proxy to send a request to an instance of "hello-world-server".
```
$ curl 127.0.0.1:4200/hello-world-server
```

You should receive a response of "Hello, world". Check that on one of the machines running an allocator, when you run `sudo docker ps` you see a container running the hello-world-server.
