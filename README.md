# PhysarumSM

PhysarumSM is published!

T. Lin, W. Zhao, I. Co, A. Chen, H. Xu and A. Leon-Garcia, "PhysarumSM: P2P Service Discovery and Allocation in Dynamic
Edge Networks," 2021 IFIP/IEEE International Symposium on Integrated Network Management (IM), 2021, pp. 304-312.

Available at: https://ieeexplore.ieee.org/document/9464074.

Or: [here](./paper.pdf).

## Getting Started
**WORK IN PROGRESS**

### 1. Bootstrapping the P2P Network
https://github.com/PhysarumSM/monitoring

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
https://github.com/PhysarumSM/service-manager

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

### 3. Registry Service
https://github.com/PhysarumSM/service-registry

The registry service, `registry-service`, is used to map service hashes to information about that service, such as its Docker hash and performance requirements. The Docker hashes are used by instances of `allocator` to pull the Docker image corresponding to a given service, and run it. The service uses `etcd` as the datastore, and thus it is a pre-requisite that must be installed. When you run `registry-service`, it will launch its own local instance of `etcd`.

To build `registry-service`:
```
$ cd service-registry/registry-service
$ go build
$ ./registry-service --new-etcd-cluster --etcd-ip <machine's IP address> -bootstrap <bootstrap-p2p-addr>
```

When running `registry-service`, you must provide:
1. The IP address which `etcd` will use to listen/advertise for peers/clients. You probably want to use the IP address of the machine you are running on.
2. The P2P address of at least one bootstrap node using the `-bootstrap` flag (the flag can be specified multiple times to specify multiple bootstraps).

To list the full set of options: `$ ./registry-service -h`

Run at least 1 instance of `registry-service` somewhere in the network. Will be used later to register a new microservice with the system.

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
https://github.com/PhysarumSM/demos

```
$ cd demos/helloworld/helloworldserver
$ go build
```

Server that responds to requests with "Hello, world". The next few steps will go over how to automatically deploy to the network. For now, you can test it out by runnning `./helloworldserver <port>` where `<port>` is the port the server will listen on. `curl 127.0.0.1:<port>` should return "Hello, world".

### 6. Containerize microservice and add to registry-service

A tool, `registry-cli`, is provided to automatically containerize you microservice alongside an instance of `proxy` that runs within the container's network namespace. It will also register the microservice with `registry-service`. You don't need to know how `proxy` works for this demo, but you can find more details in the service-manager repo.

First we need a json configuration file. You can find an example for the helloworldserver under demos/helloworld/helloworldserver called service-conf.json.

```
{
    "NetworkSoftReq": {
        "RTT": 500
    },
    "NetworkHardReq": {
        "RTT": 1000
    },
    "CpuReq": 0,
    "MemoryReq": 0,

    "DockerConf": {
        "From": "ubuntu:16.04",
        "Copy": [
            ["helloworldserver", "."]
        ],
        "Run": [],
        "Cmd": "./helloworldserver $SERVICE_PORT"
    }
}
```

The first 2 fields dictate network requirements the microservice has when making requests to other services. The next 2 fields dictate compute resources the microservice needs when allocating a new instance of itself. Note the values of these fields are just an example; they aren't really needed in this case since helloworldserver doesn't send requests and should take hardly any CPU/memory. The DockerConf section provides instructions for building the Docker image. It defines what base image to use, which files to copy into the image, and what command to user to launch the microservice. For details on all fields, see the service-registry repo.

Note the special environment variable SERVICE_PORT (there also exists PROXY_PORT and PROXY_IP variables). These variables will be set dynamically when a new container is spun up. PROXY_PORT is the port that the proxy should listen on. PROXY_IP is the IP address of the machine that the proxy/microservice will be running on. SERVICE_PORT is the port that the microserivce should listen on.

Next, on your DockerHub, create a repo named hello-world-server. We can now use the registry-cli to build the image, push to DockerHub, and register with registry-service.

```
$ cd service-registry/registry-cli
$ go build
$ ./registry-cli --bootstrap <P2P bootstrap addr> add --dir <path to helloworldserver> service-conf.json <DockerHub username>/hello-world-server:1.0 hello-world-server
```
To run registry-cli, provide the P2P address of at least 1 bootstrap node. We are running the "add" command which takes 3 mandatory arguments.

1. The config file, `service-conf.json`.
2. What you want to name the image, `<DockerHub username>/hello-world-server:1.0`. This should be the full name used to push to DockerHub, ie. `<DockerHub username>/<repo>:<tag>`.
3. The name used to register this service with registry-service, `hello-world-server`.

The --dir option should be the path to demos/helloworld/helloworldserver, which is where it looks for files listed in the service-conf.json "Copy" field, ie. the helloworldserver binary.

To list the full set of options: `$ ./registry-cli -h` and `$ ./registry-cli <command> -h`.

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
