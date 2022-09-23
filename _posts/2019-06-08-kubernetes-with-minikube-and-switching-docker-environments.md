---
title: Kubernetes with Minikube and switching Docker environments
date: 2019-06-08 14:00:00
categories: [kubernetes,docker]
tags: [kubernetes,minikube,docker]
---

![Container Graffiti](/assets/img/2019/08/container_astronaut.jpeg)

The popularity of Kubernetes as a container orchestration service has been growing at an amazing rate. Developers are flocking to the platform and taking advantage of the increase in the speed of their development deployments.

Minikube is a tool that helps you run Kubernetes locally. It runs as a single-node Kubernetes cluster on your machine so you can use it to run your software which will use the same/or similar configuration to run in production.

With Kubernetes you run Docker containers within your pods. These containers are started by running Docker images. These images can be pulled from public repositories. But what happens when you are busy developing your software locally and need quicker iterations and not have to push the image changes to a public repository so that it can be pulled from that public repository just to start up your container on a Kubernetes pod running locally.

Because Minikube runs in a VM on your machine, it has Docker and a local repository installed. So all you really need to do is build your docker images locally and tag them inside of the Minikube VM.

Minikube has no way to access your local Docker environment or repository. Luckily there is an easy way to switch context between your Minikube and local environments.

You can find out more information about installing and starting Minikube [here](https://kubernetes.io/docs/setup/minikube/).

To switch your Docker context to using Minikube you run the following command from your terminal:

```bash
#!/usr/bin/env bash
eval $(minikube docker-env)
```

Now all your Docker commands will be executed in the Minikube VM. Now all you need to do is build your image and tag it to be available when deploying to your Minikube Kubernetes instance.

```bash
#!/usr/bin/env bash

# Build docker image with no-cache flag
docker build --no-cache -t my-super-app .

# Tag it as a version
docker tag my-super-app:latest my-super-app:1.0.0
```

Now you can deploy the image to your local Minikube environment.

Add the following to a file called `deployment-my-super-app.yaml`:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-super-app
spec:
  selector:
    matchLabels:
      app: my-super-app
  template:
    metadata:
      labels:
        app: my-super-app
    spec:
      containers:
      - name: app
        image: my-super-app:1.0.0
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

```bash
#!/usr/bin/env bash
kubectl apply -f ./deployment-my-super-app.yaml
```

The important thing here is that you set the `imagePullPolicy: Never` ; This assumes that the image is available locally and will make no attempt to pull the image from a remote location.

Your pod should be running happily. When you make a code change and are ready to test out the changes in the container, you can just rebuild the image using the Docker build with no cache option. Then you can either just delete the deployment and apply it again; or just delete the pod so it will create a new one for you; or you can scale the deployment to 0 and then back up to 1, or your desired amount of replicas.

```bash
#!/usr/bin/env bash
kubectl scale --replicas=0 deployment/my-super-app
kubectl scale --replicas=1 deployment/my-super-app
```

Once you are done with your development and you would like to switch back to your original context, you can do this by running the following:

```bash
#!/usr/bin/env bash
eval $(docker-machine env -u)
```
You local Docker context would now be restored and you can happily run your local docker images again.

## Bonus

Minikube can be quite resource intensive on your machine. You can tweak the resource usage settings in VirtualBox, or you can use some simple commands to tweak your environment. Sometimes you might want to dump everything from your Minikube environment and start over. Resetting your Minikube environment and changing resources can be done with the following commands:

```bash
#!/usr/bin/env bash

set -ex

minikube config set cpus 4
minikube config set memory 4096
minikube config view
minikube delete || true
minikube start --vm-driver ${1-"virtualbox"}
```

Medium Article: [https://craignewtondev.medium.com/kubernetes-with-minikube-and-switching-docker-environments-4d965d00eb26](https://craignewtondev.medium.com/kubernetes-with-minikube-and-switching-docker-environments-4d965d00eb26)