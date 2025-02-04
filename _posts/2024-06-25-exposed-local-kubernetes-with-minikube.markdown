---
title: Exposing a local Kubernetes cluster with Minikube
date: 2024-06-25
categories: [DevOps, Kubernetes]
tags: [Kubernetes, Minikube, Ingress, DNS, Networking, VirtualBox]
---

I'm far from being a Kubernetes expert, but I'm dedicated to learning more about it, and for me, the best way to learn is by doing. It's easy to get a local Kubernetes cluster up and running, there are many tools available to help you with that, and although I'm using [Minikube](https://minikube.sigs.k8s.io/docs/) in this post, most of the content here can be applied to other tools.

To make you cluster reachable, you need a DNS pointing to the IP address of your modem/router. If you don't have a static IP, you can use a dynamic DNS service like [No-IP](https://www.noip.com/). I have a TP-Link router that provides its own dynamic DNS service, so I'm using it.

I also have a registered domain. To point my domain to the dynamic DNS provided by my router, I created a CNAME record that points to this dynamic DNS domain:

```bash
$ dig +short dev.fnix.com.br
klabs.tplinkdns.com.
186.218.192.64
```

I'm not worried about the security of my cluster, because this is a lab environment, so to mitigate the risks, I'm using [VirtualBox](https://www.virtualbox.org) to run my cluster in an isolated virtual machine. I also configured the network to use a bridged adapter, so the VM has its own IP address on my local network, allowing me to configure the router to forward the traffic of specific ports (80 and 443 for now) to the VM.

Basically, the configuration is this one:

dev.fnix.com.br -> my ISP router -> my tp-link router -> my VM running Minikube

When the user access `dev.fnix.com.br`, the request goes to my ISP router, which I configured to forward the ports 80 and 443 to the IP address of my TP-Link router. I also configured the TP-Link router to forward the traffic on these ports to the IP address of my VM, where Minikube is running.

## Minikube

On my VM, I created a folder called `minikube` to be used as my sandbox. Inside this folder, I installed minikube and kubectl using [asdf](https://asdf-vm.com/):

```bash
$ asdf plugin add minikube
$ asdf plugin add kubectl
$ asdf install minikube latest # on this date, version 1.33.1
$ asdf install kubectl latest # on this date, version 1.30.1
$ asdf local minikube latest
$ asdf local kubectl latest
```

I started Minikube with the following command:

```bash
$ minikube start
```

The output should be something like this:

```bash
😄  minikube v1.33.1 on Arch 24.0.2 (vbox/amd64)
✨  Automatically selected the docker driver
📌  Using Docker driver with root privileges
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.44 ...
🔥  Creating docker container (CPUs=2, Memory=3900MB) ...
🐳  Preparing Kubernetes v1.30.0 on Docker 26.1.1 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

Awesome, now we need to run something in our cluster, so we can be sure that it's working. The official Kubernetes documentation has a [tutorial](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/) that shows how to expose an application using Ingress. I'm going to use it to test my setup.

You can read the tutorial for more details, but it's basically the following:

```bash
$ minikube addons enable ingress
$ kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0
$ kubectl expose deployment web --type=NodePort --port=8080
```

This will enable the Ingress addon, create a deployment with a simple application, and expose it using a NodePort service. Now, we need to create an Ingress resource to be able to direct the traffic to this service:

```yaml
# example-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: dev.fnix.com.br
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080
```

And apply it:

```bash
$ kubectl apply -f example-ingress.yaml
```

Now, we are ready to expose our application to the world. To do this we can use the `minikube tunnel` command:

```bash
$ minikube tunnel --bind-address "0.0.0.0"
```

This command will create a tunnel to the cluster, allowing the traffic to reach the services running in it. Now, we can access our application using the domain we configured in the DNS from any point in the world:

```bash
$ curl dev.fnix.com.br
Hello, world!
Version: 1.0.0
Hostname: web-56bb54ff6d-r6pns
```

That's it! We have a local Kubernetes cluster running in a VM, exposed to the world. With this setup, we can share our work with others and test features that depends on access from the internet. Some services, like [Let's Encrypt](https://letsencrypt.org/), need to reach your cluster. Other more complex services, needs domains/subdomains to be reachable from the internet, like [Rancher](https://rancher.com/), [GitLab](https://gitlab.com/), etc.