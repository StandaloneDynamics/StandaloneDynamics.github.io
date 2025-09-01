---
title: "Deploying Apps to Microk8s on a Raspberry Pi - Part 2"
date: "2024-01-20"
description: "Deploying Apps to Microk8s on a Raspberry Pi"
tags: ["Microk8s", "Docker", "Kubernetes", "Raspberry Pi"]
draft: true
---


# FastAPI + VueJS

In [Part 1](../deploy_microk8s_part_1) of this guide, we setup a Microk8s instance and deployed a rust web application. In part 2, we are going to deploy a web application that has a separate frontend and backend and see how they can communicate with one another.

The backend uses the [FastAPI](https://fastapi.tiangolo.com/) framework.

The frontend uses the [Vuejs](https://vuejs.org/).


## Docker


### Backend Image

The backend application consists of just 2 endpoints.

![api endpoints](/fastapi.png)



Here is how the DockerFile looks like.

```
FROM python:3.11-bookworm

WORKDIR /code

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY ./app /code/app

EXPOSE 8000

CMD ["fastapi", "run", "app/main.py", "--port", "8000"]
```

We'll create an image called `weight-backend`, we'll build it by running

```
docker buildx build -t development.local:32000/weight-backend:latest --load --platform linux/arm64 .
```

Once the image is built we can push it to our private registry.

```
docker push development.local:32000/weight-backend:latest
```

### Frontend Image
Lets create a DockerFile for our frontend image and build it.

```
FROM node:18

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

RUN npm install -g http-server

EXPOSE 5000

# Command to run the application
CMD ["http-server", "-p", "5000", "dist"]

```
We'll use the the simple `http-server` to serve our application as it will be enough for our situation.

We'll call this image `weight-frontend`.


```
>>> docker buildx build -t development.local:32000/weight-frontend:latest --load --platform linux/arm64 .
```

Lets push

```
>>> docker push development.local:32000/weight-frontend:latest
```


## Manifests

On the development server lets create a folder `apps/weight` to place our kubernetes files in, this folder is different to our `weight-tracker` folder which contains our rust application.

### Storage

One thing we are going to change from the rust based application is that we are going to use the `sqlite` database instead of `postgresql`. This change means that we need a place to store the sqlite database file, which we'll call `weight-tracker.db`. 

We cannot store the database in the pods as they are ephemeral, meaning that when you restart a pod any data that was stored there will be deleted. 

We need an alternative solution. In the docker space, if you want to persist data you'd have to use [docker volumes](https://docs.docker.com/engine/storage/volumes/), that idea can be extended to kubernetes, the terminology used is [PersistantVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) also called `storage classes` and [PersistantVolumeClaims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes).

How I like to think about the 2 concepts is - one manages the storage (PV) and one asks for a bit of storage (PVC) that will be used by the containers

In microk8s there is an addon called [Hostpath Storage](https://microk8s.io/docs/addon-hostpath-storage) which is a certain type of storage, which will allow us to create the necessary storage on the development server.

The are various other types of [storage](https://kubernetes.io/docs/concepts/storage/storage-classes/) options in the kubernetes space.

```
>>> microk8s enable hostpath-storage
```

One very important thing to note, this will only work for a single node cluster, If you have a multi node cluster (multiple raspberry pis) something like [CSI for NFS](https://microk8s.io/docs/how-to-nfs) would be more appropriate.

Let create the necessary storage manifests for the backend container.

`backend-storage.yaml`
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-hostpath
provisioner: microk8s.io/hostpath
reclaimPollicy: Deleted
volumeBindingMode: WaitForFirstConsumer
parameters:
  pvDir: ~/app/database
```
So, we've created a storage volume that's 1GB in the `~/app/database` directory on the server. The `volumeBindingMode` tells us when provisioning of the storage should occur, in our case `WaitForFirstConsumer` means wait until the pod schedules it.
We can apply this manifest.

```
>>> kubectl apply -f backend-storage.yaml
```
We can check if our storage has been created by 

```
>>> kubectl get storageclass
```

We now we what to claim(use) a portion of storage for our weight application, lets create its manifest.

`backend-pvc.yaml`
```
apiVersion: apps/v1
kind: PersistentVolumeClaim
metadata:
  name: weight-pvc
spec:
  storageClassName: database-hostpath
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50mb
```
Since we will only be storing the database file 50MB should be enough for a very long time, applying

```
>>> kubectl apply -f backend-pvc.yaml
```
we can check it has being created.

```
>>> kubectl get pvc
```

The storage has not been provisioned at this stage, we've just specified a claim that we are going to use. So now that is setup we can start creating our deployment manifest.


### Deployment

`backend-deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weight-backend-deployment
  labels:
    app: weight-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: weight-backend
  template:
    metadata:
      labels:
        app: weight-backend
    spec:
      volumes:
        - name: fastapi-store
          persistantVolumeClaim:
            claimName: weight-pvc
      containers:
        - name: weight-backend
          image: development.local:32000/weight-backend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          volumeMounts:
            - mountPath: /code/app/data
              name: fastapi-store
```

So in `spec.volumes` we've defined a volume that we are going to use, and within the container, we are going to mount it on `/code/app/data`, remember in our Dockerfile our app runs in the directory `/code/app`. So when the application starts up the sqlite database, `weight-tracker.db` will be placed in the `data` directory.


`frontend-deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weight-frontend-deployment
  labels:
    app: weight-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: weight-frontend
  template:
    metadata:
      labels:
        app: weight-frontend
    spec:
      containers:
        - name: weight-frontend
          image: development.local:3200/weight-frontend:latest
          ports:
            - containerPort: 5000

```

Applying our deployment manifests

```
>>> kubectl apply -f backend-deployment.yaml
>>> kubectl apply -f frontend-deployment.yaml
```

We can check that they have being deployed via

```
>>> kubectl get pods
```





### Service

Lets also create our service manifests.

`backend-service.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: weight-backend-service
spec:
  selector:
    app: weight-backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

`frontend-service.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: weight-frontend-service
spec:
  ports:
    - port: 80
      targetPort: 5000
  selector:
    app: weight
```

lets apply these 2 manifests via

```
>>> kubectl apply -f backend-service.yaml

>>> kubectl apply -f frontend-service.yaml
```

Running `kubectl get services`, we see



One thing we need to think now is how is our frontend app going to communicate with our backend, recall that a service  exposes the pods to the network, so if we run `curl ip address` of the service, we should be able to hit the backend api and frontend ui. One feature of kubernetes is that it creates DNS records for services, which is the name given to the service. So instead of using the  IP address of the service which could change, we can use the name of the service which will be `weight-backend-service`. In the vuejs app we have to remember to change the baseURL to point to this url, eg: `http://weight-backend-service`


### Ingress

We only need to make the frontend of the app accessible from outside the cluster and not the backend.
We can go ahead an create local dns entry in our `pihole`. We will call this application simply weight.

![weight-dns](/weight-dns.png)


`ingress.yaml`
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: weight-frontend-ingress
spec:
  rules:
    - host: weight.development.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: weight-frontend-service
                port:
                  number: 80
```

Applying

```
>>> kubectl apply -f weight-frontend-ingress.yaml
```


We should now be able to access our application from the browser.


## Conclusion

So in this guides we've gone through the process of setting up a Microk8s instance and deployed a couple of applications on it. This has allowed us to get a better understanding of the Kubernetes ecosystem and the ideas around it.
