This is a fork of https://gitlab.com/cryptexlabs/public/development/minikube-ingress-dns/

# Minikube Ingress DNS

DNS service for ingress controllers running on your minikube server

## Overview

### Problem
When running minikube locally you are highly likely to want to run your services on an ingress controller so that you don't have to use minikube tunnel or NodePorts to access your services. While NodePort might be ok in a lot of circumstances in order to test some features an ingress is necessary. Ingress controllers are great because you can define your entire architecture in something like a helm chart and all your services will be available. There is only 1 problem. That is that your ingress controller works basically off of dns and while running minikube that means that your local dns names like `local.service` will have to resolve to `$(minikube ip)` not really a big deal except the only real way to do this is to add an entry for every service in your `/etc/hosts` file. This gets messy for obvious reasons. If you have a lot of services running that each have their own dns entry then you have to set those up manually. Even if you automate it you then need to rely on the host operating system storing configurations instead of storing them in your cluster. To make it worse it has to be constantly maintained and updated as services are added, remove, and renamed. I call it the `/ets/hosts` pollution problem.

### Solution
What if you could just access your local services magically without having to edit your `/etc/hosts` file? Well now you can. This project acts as a DNS service that runs inside your kubernetes cluster. All you have to do is install the service and add the `$(minikube ip)` as a DNS server on your host machine. Each time the dns service is queried an API call is made to the kubernetes master service for a list of all the ingresses. If a match is found for the name a response is given with an IP address as the `$(minikube ip)` which was provided when the helm chart was installed. So for example lets say my minikube ip address is `192.168.99.106` and I have an ingress controller with the name of `myservice.local` then I would get a result like so: 

```text
#bash:~$ nslookup myservice.test
Server:		192.168.99.106
Address:	192.168.99.106#53

Non-authoritative answer:
Name:	myservice.test
Address: 192.168.99.106
```

## Installation

### Start minikube
```
minikube start
```

### Install the kubernetes resources
```bash
kubectl apply -f app/dns-server/k8s/
```

### Add the minikube ip as a dns server

#### Mac OS
Create a file in /etc/resolver/minikube-profilename-test
```
domain test
nameserver 192.168.99.169
search_order 1
timeout 5
```
Replace `192.168.99.169` with your minikube ip

#### Linux

TODO

## Testing

### Add the test ingress
```bash
kubectl apply -f example/
```

### Validate DNS queries are returning A records
```bash
nslookup hello-john.test $(minikube ip)
nslookup hello-jane.test $(minikube ip)
```

### Validate domain names are resolving on host OS
```bash
ping hello-john.test
```
Expected results:
```text
PING hello-john.test (192.168.99.169): 56 data bytes
64 bytes from 192.168.99.169: icmp_seq=0 ttl=64 time=0.361 ms
```
```bash
ping hello-jane.test
```
```text
PING hello-jane.test (192.168.99.169): 56 data bytes
64 bytes from 192.168.99.169: icmp_seq=0 ttl=64 time=0.262 ms
```

### Curl the example server
```bash
curl http://hello-john.test
```
Expected result:
```text
Hello, world!
Version: 1.0.0
Hostname: hello-world-app-557ff7dbd8-64mtv
```
```bash
curl http://hello-jane.test
```
Expected result:
```text
Hello, world!
Version: 1.0.0
Hostname: hello-world-app-557ff7dbd8-64mtv
```

## Multiple DNS servers

### Problem

When running multiple minikube clusters, it might be desirable to activate Ingress DNS on all clusters and ask all DNS servers at once. Configuring the system to ask all minikube DNS servers is out of scope of this description, you might need a local DNS service like dnsmasq (Linux) or Acrylic DNS Proxy (Windows) to do that. Problem will arise when multiple DNS servers are queried at the same time - the first response wins. If the first response is the not-found (NoData) response, it is forwarded to the client, which is not the intended behavior.

### Solution

There is a configuration value `dns-nodata-delay-ms`, which adds delay to the not-found (NoData) response. Simply edit the ConfigMap, set a non-zero value like `"20"` (meaning 20 milliseconds) and wait until it is applied:

```bash
kubectl edit -n kube-system configmap minikube-ingress-dns
```

The update takes some time, so you can check if the value was applied in logs, you should see message `Updated NoData packet delay to 20 ms`:

```bash
kubectl logs -n kube-system kube-ingress-dns-minikube
```

or you can check the actual value which is visible to the Pod itself:

```bash
kubectl exec -n kube-system kube-ingress-dns-minikube -- /bin/sh -c "cat /config/dns-nodata-delay-ms"
```

## Known issues

### .localhost domains will not resolve on chromium
.localhost domains will not correctly resolve on chromium since it is used as a loopback address. Instead use .test, .example, or .invalid

### Mac OS

#### mDNS reloading
Each time a file is created or a change is made to a file in `/etc/resolver` you will need to run the following to reload Mac OS mDNS resolver.
```bash
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
```
