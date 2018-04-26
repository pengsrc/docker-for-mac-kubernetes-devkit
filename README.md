# Development Toolkit for Kubernetes on Docker for Mac

Docker for Mac (on Edge channel) includes a local Kubernetes cluster which is very delightful for test and development. Refer to the official document ([https://docs.docker.com/docker-for-mac/#kubernetes](https://docs.docker.com/docker-for-mac/#kubernetes)) to know how to get it up and running.

If you are using Kubernetes on Docker for Mac, some scripts in this repository might be helpful.

## Table of Content

- [Pod/Docker Network Access](#pod-docker-network-access)
- [Nginx Ingress Controller](#nginx-ingress-controller)

## <a name="pod-docker-network-access">Pod/Docker Network Access</a>

Because the Docker for Mac containers are actually running in a VM powered by
HyperKit, you can't directly have interactions with the containers. More details here, _[Docker for Mac - Networking - Known limitations, use cases, and workarounds](https://docs.docker.com/docker-for-mac/networking/#known-limitations-use-cases-and-workarounds)_.

To solve this problem, run an OpenVPN server container inside the VM with `host` network mode, then you can reach the containers with its internal IP. You can run the OpenVPN server with docker-compose or on Kubernetes.

Off course you can follow the docker-compose approach without Kubernetes.

Generally, it works like this:

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
dirPaths:
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

## <a name="nginx-ingress-controller">Nginx Ingress Controller</a>

In most times, a Kubernetes ingress controller is needed to manage all traffic, but there's no cloud available for a Mac.

If you define your `service` type as `LoadBalancer` in Kubernetes, Docker for Mac will open a port on host machine. So we can deploy the [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx) to serve and dispatch requests.

First of all, stop anything that listing on port 80 or 443.

``` Bash
$ kubectl apply -f ingress-nginx/namespaces
namespace "ingress-nginx" created

$ kubectl apply -f ingress-nginx/configmaps
configmap "nginx-configuration" created
configmap "tcp-services" created
configmap "udp-services" created

$ kubectl apply -f ingress-nginx/deployments
deployment "default-http-backend" created
deployment "nginx-ingress-controller" created

$ kubectl apply -f ingress-nginx/services
service "default-http-backend" created
service "nginx-ssl" created
service "nginx" created
```

Now nginx ingress controller is listing on port 80 and 443, visit http://127.0.0.1 will see the default HTTP backend.

``` Bash
$ curl http://127.0.0.1
default backend - 404
```

Check the ingress controller.

``` Bash
$ kubectl get all -n ingress-nginx
NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/default-http-backend       1         1         1            1           21m
deploy/nginx-ingress-controller   1         1         1            1           21m

NAME                                     DESIRED   CURRENT   READY     AGE
rs/default-http-backend-55c6c69b88       1         1         1         21m
rs/nginx-ingress-controller-579f8bf799   1         1         1         21m

NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/default-http-backend       1         1         1            1           21m
deploy/nginx-ingress-controller   1         1         1            1           21m

NAME                                     DESIRED   CURRENT   READY     AGE
rs/default-http-backend-55c6c69b88       1         1         1         21m
rs/nginx-ingress-controller-579f8bf799   1         1         1         21m

NAME                                           READY     STATUS    RESTARTS   AGE
po/default-http-backend-55c6c69b88-9rcr7       1/1       Running   0          21m
po/nginx-ingress-controller-579f8bf799-4pfjk   1/1       Running   0          21m

NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
svc/default-http-backend   ClusterIP      10.110.153.121   <none>        80/TCP          21m
svc/nginx                  LoadBalancer   10.100.205.59    localhost     80:31764/TCP    21m
svc/nginx-ssl              LoadBalancer   10.108.87.129    localhost     443:30592/TCP   21m
```

## License

The Apache License (Version 2.0, January 2004).
