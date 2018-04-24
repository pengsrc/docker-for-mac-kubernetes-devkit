# Development toolkit for Kubernetes on Docker for Mac

Docker for Mac (on Edge channel) includes a local Kubernetes cluster which is very delightful for test and development. Refer to the official document ([https://docs.docker.com/docker-for-mac/#kubernetes](https://docs.docker.com/docker-for-mac/#kubernetes)) to know how to get it up and running.

If you are using Kubernetes on Docker for Mac, some scripts in this repository might be helpful.

## Pod/Docker Network Access

Because the Docker for Mac containers are actually running in a VM powered by
HyperKit, you can't directly have interactions with the containers. More details here, _[Docker for Mac - Networking - Known limitations, use cases, and workarounds](https://docs.docker.com/docker-for-mac/networking/#known-limitations-use-cases-and-workarounds)_.

To solve this problem, run an OpenVPN server container inside the VM with `host` network mode, then you can reach the containers with its internal IP. You can run the OpenVPN server with docker-compose or on Kubernetes.

Off course you can follow the docker-compose approach without Kubernetes.

Generally, it Works like this:

``` Text
Mac <-> Tunnelblick <-> socat/service <-> OpenVPN Server <-> Containers
```

### Prepare

1. Install [`Tunnelblick`](https://tunnelblick.net/downloads.html) (an open source GUI OpenVPN client for Mac).

2. Change into the `docker-for-mac-openvpn` directory.

### Run OpenVPN on Kubernetes (Approach #1)

1. Install [`helm`](http://helm.sh) (the package manager for Kubernetes).

2. Create local values file at `local/values.yaml` and specify local dirs.

``` Text
dirPath:
  # The project dir.
  data: /tmp/docker-for-mac-kubernetes-devkit/docker-for-mac-openvpn
  # Local dir to hold generated files.
  local: /tmp/docker-for-mac-kubernetes-devkit/docker-for-mac-openvpn/local
  # Local dir to hold generated server configs.
  configs: /tmp/docker-for-mac-kubernetes-devkit/docker-for-mac-openvpn/local/configs
```

3. Run the OpenVPN server.

``` Bash
$ helm install -n docker-for-mac -f local/values.yaml .
```

### Run OpenVPN server with docker-compose (Approach #2)

Run the OpenVPN server, it'll generate certificates and configurations at the first time, maybe a little slow.

``` Bash
$ # Run
$ docker-compose up -d
$ # Follow logs
$ docker-compose logs -f
```

### Configure Client

Now, you will get the client config file at `./local/docker-for-mac.ovpn`. Add the subnets that you want to reach at bottom of the client config like below, and connect to the local OpenVPN server.

``` Config
route 172.16.0.0 255.255.0.0
route 10.96.0.1 255.240.0.0
```

### Test Network

Run a container and access to it directory with it's IP address.

``` Bash
$ # Start Nginx
$ docker run --rm -it nginx

$ # Find out the IP address
$ docker inspect `docker ps | grep nginx | awk '{print $1}'` | grep '"IPAddress"'
"IPAddress": "172.16.0.11",

$ # Visit
$ curl http://172.16.0.11
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

## License

The Apache License (Version 2.0, January 2004).
