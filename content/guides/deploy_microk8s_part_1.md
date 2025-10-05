---
title: "Deploying Apps to Microk8s on a Raspberry Pi - Part 1"
date: "2024-01-20"
description: "Deploying Apps to Microk8s on a Raspberry Pi"
tags: ["Microk8s", "Docker", "Kubernetes", "Raspberry Pi", "Rust"]
draft: false
---

# Introduction

In this guide we'll be going through the steps of setting up a [Microk8](https://microk8s.io/) instance and deploying some applications to it. The aim is to get an better understanding of [Kubernetes](https://kubernetes.io/) and figuring out what's involved in getting applications running on it.
Microk8s is a lightweight version of Kubernetes meant to run on smaller devices such as raspberry pis'. It also comes with addons which allow you to add features as needed.

This guide will be broken into 2 parts

* Part 1, we will setup Microk8s and deploy a [Rust](https://www.rust-lang.org/) based application and use the [Rocket](https://rocket.rs/) web framework.

* [Part 2](../deploy_microk8s_part_2), we will deploy an application which has a seprate backend and frontend. The backend will be using [FastApi](https://fastapi.tiangolo.com/) while the frontend will be [Vue](https://vuejs.org/) based.

In each of the deployments we will come accross various services of kubernetes which will help us learn more about it. 


Creating the applications is not the focus of this guide, just the setup of the Microk8 instance and the configurations needed to deploy them.

The deployment of the web application will not involve any automation tools (CI/CD/helm), rather we'll manually deploy them, then create a bash script to somewhat automate the process..


# Infrastructure Setup

The setup for this guide looks as follows.

* Rasberry Pi 4  - Used to run Microk8s with our applications with a hostname of `development.local`.
* Raspberry Pi 2 - Used for local DNS using PiHole, hostname `dns.local`, which will come in handy later.

All the above raspberry pi's are running Ubuntu 24.04 LTS and are on the same internal network.


## Install

Before Microk8s can be setup on our `development.local` machine, control groups (`cgroups`) need to be enabled and certain kernel modules installed on the raspberry pi.

* Add the config variable `cgroup_enable=memory cgroup_memory=1` to the `/boot/firmware/cmdline.txt` file.

If you have a kernel version <= 6.7, you'll need to run the below command.

* `>>> sudo apt install linux-modules-extra-raspi`

* Reboot your raspberry pi to make sure your changes have taken place.

We can now install Microk8s using the following

```
>>> sudo snap install microk8s --classic
```

This will install the latest stable version, the stable versions follows the release cycle of the kubernetes project.

We then add our user to the microk8s group and set permissions on the `~/.kube` config directory.

```
>>> sudo usermod -a -G microk8s $USER
```
 where `$USER` is the currently logged in user.

```
>>> sudo chown -f -R $USER ~/.kube
```
Logout then login to apply the permissions.

So now that we have Microk8s installed, lets check if it's up and running.

```
>>> microk8s kubectl get services
```

This should return a table with any services that are currently running. This shows us that our instance is running and working correctly.

| Name       | Type      | CLUSTER-IP   | EXTERNAL-IP | PORT(s) | AGE   |
|:-----------|:----------|:-------------|:------------|:--------|:------|
| Kubernetes | ClusterIP | 10.152.183.1 | <none>      | 443/TCP | 4m11s |

Your Cluster-IP address could be different.


We can shorten these microk8s commands by creating an alias in our `~/.bashrc` file.

```
>>> alias kubectl='microk8s kubectl'
```

Reload the bash settings via, 
```
>>> source ~/.bashrc
```
Now we can use the alias as such

```
>>> kubectl get services
```
This is how the commands look when using actual Kubernetes, nice.


## Image Storage

We need a place to store our docker images that we are going to build. We could store them on [docker hub](https://hub.docker.com/_/registry) but for this guide we are going to take the self hosted route.

Microk8s comes with an addon called [registry](https://microk8s.io/docs/registry-images), which will allow us to do just that. So it will allow us to create our own private registry whereby we can pull and push our docker images.

To enable it:

```
>>> microk8s enable registry
```

This creates an insecure registry by default but since we are using this on our local network this will be fine.
If you are planning on using this over the internet it's recommened to make it secure. You can read more about it [here](https://microk8s.io/docs/registry-private).

An important thing to note is that the registry sits outside of Microk8s, so images will need to be pulled into it, so we have to tell Microk8s about this registry.
We have to create a [hosts.toml](https://microk8s.io/docs/registry-private#configuring-microk8s) and configure some settings

```
>>> sudo mkdir -p /var/snap/microk8s/current/args/certs.d/10.42.0.105:32000
```
Remember that out registry will be on the same machine as the Mirok8s instance, so I'm using the ip address here

Create a `hosts.toml` file in that directory

```
>>> sudo touch /var/snap/microk8s/current/args/certs.d/10.42.0.105:32000/hosts.toml
```

Add the following to that file.

```
server = "http://10.42.0.105:32000"

[host."http://10.42.0.105:32000"]
capabilities = ["pull", "resolve"]
```

We need to restart Microk8s for the changes to take affect.

This will now allow us to pull the images into Mircok8s.

## Docker

By default docker does not allow pushing images to insecure registries, which our newly created one is, so we have to explicitly allow it in the docker config file `/etc/docker/daemon.json`. This is the config file on the machine in which you will be builing and pushing images, in my case my desktop.

In the file add

```
{
  "insecure-registries" : ["development.local:32000"]
}
```

This will now allow us to push our images to our registry. You can use the `IP` address of the machine as well.
  

## DNS

When we have our applications running we want to be able to access them from other machines by using a dns name instead of an ip address.

[Pihole](https://pi-hole.net/) allows us to setup local DNS within our network so that we can use a dns name instead of the ip address.
So for instance, we will be accessing our apps via `http://<app>.development.local` where `app` will be the name of the application.

To setup local DNS in the pihole see [here](https://docs.pi-hole.net/ftldns/configfile/)


# Weight Tracker Application

The web application is a very simple one, keeps track of a persons weight. Enter value and date then plot on graph. We'll be building 2 versions of this application. The rust version, while the other will be built with FastAPI and Vuejs.

![weight-tracker](/weight-tracker.png)


## Rust Version

### Docker
Ok, lets create a docker image of our project.

We'll be using multi-stage build step in order to reduce the size of our image

Since this image will be deployed to a raspberry pi (armv8) which has a different architecture to the desktop machine that we are developing(amd64), we need to utilize multi-arch building, 

For that we need to use the [docker buildx](https://docs.docker.com/buildx/working-with-buildx/) tool which will allow us to achive that.

#### Image

Our Dockerfile will look as follows

```
FROM rust:1.84-bookworm AS builder

RUN apt update && apt upgrade -y 
RUN apt install -y g++-arm-linux-gnueabihf libc6-dev-armhf-cross gcc-arm-linux-gnueabihf

RUN rustup target add aarch64-unknown-linux-gnu
RUN rustup toolchain install stable-aarch64

WORKDIR /build
COPY src/ /build/src/
COPY Cargo.toml /build/

RUN cargo build --target aarch64-unknown-linux-gnu --release


FROM rust:1.84-slim
WORKDIR /app
COPY --from=builder /build/target/aarch64-unknown-linux-gnu/release/weight-tracker /app/
COPY templates/ /app/templates/
COPY static /app/static/


RUN chmod +x weight-tracker
EXPOSE 8000

```
A few things to note.

* We explicitly specifying what architecture we want to build for by using `--target aarch64-unknown-linux-gnu`
* weight-tracker is the name of the binary that is generated.
* For the rocket web framework the template and static files are not automaticaly included in the binary so they have to copied over to the same directory as the binary

We are now ready to build our image with the following command.

```
>>> docker buildx build -t development.local:32000/weight-tracker:latest --load --platform linux/arm64 .
```
* For the `docker buildx` command, we have specified which platform we are building for by using the `--platform linux/arm64` parameter.

* We also need to tag (`-t`) our image by using the location of where our registry is.

Once our image is built we can push it to our registry.

```
>>> docker push development.local:32000/weight-tracker:latest
```


Now that we have pushed our image, we need to pull it in. We can do this by

```
>>> microk8s ctr image pull --plain-http development.local:32000/weight-tracker:latest
```

### Manifests

Ok so now our image is in our registry, let create a kubernetes manifests files.

**There are ways to automate the creation of kubernetes files but the focus of this guide is to get a better understanding of kubernetes, so I'll be creating these manually.**

On our server we are going to create a couple of directories in `$HOME`
```
>>> mkdir -p apps/weight-tracker
```

#### Deployment


In kubernetes, a deployment manifest describes how you want your pods to be managed and ran.
In our case the pod will consist of one container.
We are going to try to keep our declaration as simple as possible.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weight-tracker-deployment
  labels:
    apps: weight-tracker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: weight-tracker
  template:
    metadata:
      labels:
        app: weight-tracker
    spec:
      volumes:
        - name: config
          configMap:
            name: weight-tracker-config
            items:
              - key: "Rocket.toml"
                path: "Rocket.toml"
      containers:
        - name: weight-tracker
          image: 10.42.0.101:32000/weight-tracker:latest
          ports:
            - containerPort: 8000
          env:
            - name: ROCKET_CONFIG
              value: /app/data/Rocket.toml
          volumeMounts:
            - name: config
              mountPath: /app/data
              readOnly: true
          command: ["./weight-tracker"]

```
So what's going on in this deployment file.

* We are have created a deployment object called `weight-tracker-deployment` specified in `metadata.name` field.
* The specification of this deployment are under `spec` field.
* `spec.replicas` indicates how many pods should be created.
* `spec.selector.matchLables` tells us how the replica should find the pods, in our case any pods with the label `weight-tracker`
* `spec.template` indicates how the Pods should be created.
* `spec.template.spec` shows the details of the container that will be ran in the Pod, this looks very similar to how a docker compose file would look like.

If you look under `spec.volumes` you'll see that we have created some sort of [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/), this will allow us to inject application configurations into the pod, we will create this object next.

#### ConfigMap

Our web framework that we used, rocket, uses a configuration file called `Rocket.toml` that is used to specify certain parameters, such as port number and database urls, so what we'll like to do is inject this config file into our pod when it starts up. This is where ConfigMaps come into play.

In the `apps/weight-tracker` directory we'll create a file called `Rocket.toml` and populate it with the parameters. We'll create a ConfigMap object via the following command.

```
kubectl create configmap weight-tracker-config --from-file=Rocket.toml
```
So here we created a ConfigMap object with the name of `weight-tracker-config` and we want to load the contents of the file from the Rocket.toml file. Out of interest we can use `kubectl describe configmap weight-tracker-config` to see how the object looks like.

```
Name:         weight-tracker-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
Rocket.toml:
----
[default]
address="0.0.0.0"

[default.databases.weight_tracker]
url="postgresql://<username>:passwordt@db.local:5432/weight_tracker"

BinaryData
====

Events:  <none>
```
So under the Data section we can see that the data is store as a key-value setup `Rocket.toml:<file contents> `.

Now that we have setup this up lets go back let take a look at our deployment file.

```
volumes:
        - name: config
          configMap:
            name: weight-tracker-config
            items:
              - key: "Rocket.toml"
                path: "Rocket.toml"
```
So here we have created a volume mount named config which uses the weight-tracker-config object we created, we then specify which key from the object should be used, in our case Rocket.toml. The `path` parameter indicates what name the file should be called when mounted.

So now lets mount the file.

```
volumeMounts:
            - name: config
              mountPath: /app/data
              readOnly: true
```

So we've specified which volume to use as there could be multiple configured and we have specified is which directory to mount this file.
So the full path of this file will be `/app/data/Rocket.toml`.

An important this to note is that if the data directory already exists with files, then those files will be removed.


Ok, lets apply our deployment file and check that it's working.

```
>>> kubectl apply -f deployment.yaml
```

We can see the status of this deployment by running
```
>>> kubectl get pods
```

| Name                                       | Ready | Status  | Restarts | Age   |
|:-------------------------------------------|:------|:--------|:---------|:------|
| weight-tracker-deployment-399b9c4d7d-jhj4s | 2/2   | Running | 0        | 1m22s |
| weight-tracker-deployment-699b9cds7d-jhj4s | 2/2   | Running | 0        | 1m20s |


Looking all good so far.
We can also check if our weight tracker application is working by looking at the logs

```
>>> kubectl logs weight-tracker-deployment-699b9c4d7d-jhj4
```
For our application output is
```
Configured for production.
    => address: 0.0.0.0
...
Rocket has launched from http://0.0.0.0:8000
```

So we can see that our web application has successfully launched.

Now need some way to access these pods over the network


#### Service
Right now we do not have a way to access our application, this is where a Kubernetes [Service](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/) will come into play.
A service allows us to expose our pods to the network by proving and endpoint we can use to connect to the pods.
A simple example is if you have separate backend pods and separate frontend pods and they need to communicate with each other, they can communicate via services.

So when a request comes to the `weight-tracker-service` that request will be mapped to the port that the pods are running on, in our case port 8000.

Lets create a very simple service declaration in `service.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: weight-tracker-service
spec:
  selector:
    app: weight-tracker
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000

```
* We tell the service which Pods to target via the `spec.selector`, remember in the deployment file we defined the name as `template.metadata.labels.app.weight-tracker`

Ok, lets apply this service object  and check that it is working.

```
>>> kubectl apply -f service.yaml
```

We can see the status of this deployment by running

```
>>> kubectl get services
```

| NAME                   | TYPE      | CLUSTER-IP     | EXTERNAL-IP | PORTS  | AGE |
|:-----------------------|:----------|:---------------|:------------|:-------|:----|
| weight-tracker-service | ClusterIP | 10.152.183.174 | <None>      | 80/tcp | 20s |


So, we've created a service of type `ClusterIP`, this means that this service is only available within the cluster and not externally.
We can check if we can reach our application by trying.

```
curl 10.152.183.174
```

The output will be the homepage of our application

```
<!DOCTYPE html>
<html>
<head>

.....
```

There are other service types such as `NodePort` which creates a static port on each of your nodes to allow for external communication, so in the scenario we could access the service via the raspberry pis `IP` address + the `port` number.

So we've got our service running, now we'd like to connect to this service externally via  http such as `http://weight-tracker.development.local`

#### Ingress

We'll be using [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) object to manage external access.
This comes as an addon in microk8s, so we have to install it first.

```
>>> microk8s enable ingress
```
This will essentially create an ingress controller which will allow nginx to act as a reverse proxy and connect us to our `weight-tracker-service`


So lets create an ingress file to allow accces to our `weight-tracker-service`

Lets first setup a dns name for our application, we can do this on a [PiHole](https://pi-hole.net/).
We'll call it `weight-tracker.development.local` this will point to the ip of our `development.local` server.

On the Pihole, it will look something like this

![screenshot](/dns.png)




Ok. lets create our ingress file `ingress.yaml`.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: weight-tracker-ingress
spec:
  rules:
    - host: weight-tracker.development.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: weight-tracker-service
                port:
                  number: 80
```

So ingress will listen to all requests coming into `http://weight-tracker.development.local` and route the traffic to the appropriate service defined in `backend.service` on `port.number=80`


Ok, lets apply this component and check that its working
```
>>> kubectl apply -f ingress.yaml
```

We can see the status of this deployment by running
```
>>> kubectl get ingress
```

| NAME                   | CLASS  | HOSTS                            | ADDRESS   | PORTS | AGE |
|------------------------|--------|----------------------------------|-----------|-------|-----|
| weight-tracker-ingress | public | weight-tracker.development.local | 127.0.0.1 | 80    | 20s |


#### Deployed

So when we try to access our weight tracking application via `http://weight-tracker.development.local` we should get presented with our application

show screenshot here.

So we have managed to deploy our rust application to Microk8s, in the next section we'll be deploying a django backend application with a vuejs frontend.

In [Part 2](./deploy_microk8s_part2.md) we are going to deploy a FastApi and Vuejs application.

