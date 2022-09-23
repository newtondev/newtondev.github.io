---
title: How to fix — Kubernetes namespace deleting stuck in Terminating state
date: 2019-07-12 12:00:00
categories: [kubernetes]
tags: [kubernetes,troubleshooting]
---

![Danger - do not proceed beyond this point](/assets/img/2019/09/danger.jpeg)

So AWS launched their hosted Kubernetes called EKS (Elastic Kubernetes Service) last year and we merrily jumped onboard and deployed most of our services in the months since then. Everything was looking great and working amazing until I started doing some cleanup on the platform. I am sure that it was possibly my fault for adding deployments, removing deployments, moving nodes around, terminating nodes, etc. over a period of time, I pretty much scrambled its marbles.

I removed everything from a namespace I had created called "logging" as I was moving all the resource to a different namespace called "monitoring". I then proceeded to delete the namespace using `kubectl delete namespace logging`. Everything looked great and when I listed the namespaces and it was showing the state of that namespace as `Terminating`. I proceeded with my day to day DevOps routine and came back later to find the namespace still stuck in the `Terminating` state. I ignored it and went and had a great weekend. 

Coming back on Monday, that same namespace was still there, stuck on `Terminating`. Frustrated I turned to Google and started looking if anyone had the same issue. There were quite a few results, tried most of them to no avail until I stumbled on this issue: [https://github.com/kubernetes/kubernetes/issues/60807](https://github.com/kubernetes/kubernetes/issues/60807). I tried most of the recommendations also to no avail until I started mixing and matching some of the commands, and then finally, I got rid of that namespace that was stuck in the `Terminating` state. I know, I could have left it there and not bothered with it, but my OCD kicked in and I wanted it gone ;)

So it turns out I had to remove the finalizer for `kubernetes`. But the catch was not to just apply the change using `kubectl apply -f`, it had to go via the cluster API for it to work.

Here are the instructions I used to delete that namespace:

## Step 1: Dump the descriptor as JSON to a file

```shell
kubectl get namespace logging -o json > logging.json
```

Open the file for editing:
```json
{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        "creationTimestamp": "2019-05-14T13:55:20Z",
        "labels": {
            "name": "logging"
        },
        "name": "logging",
        "resourceVersion": "29571918",
        "selfLink": "/api/v1/namespaces/logging",
        "uid": "e9516a8b-764f-11e9-9621-0a9c41ba9af6"
    },
    "spec": {
        "finalizers": [
            "kubernetes"
        ]
    },
    "status": {
        "phase": "Terminating"
    }
}
```

Remove `kubernetes` from the `finalizers` array:
```json
{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        "creationTimestamp": "2019-05-14T13:55:20Z",
        "labels": {
            "name": "logging"
        },
        "name": "logging",
        "resourceVersion": "29571918",
        "selfLink": "/api/v1/namespaces/logging",
        "uid": "e9516a8b-764f-11e9-9621-0a9c41ba9af6"
    },
    "spec": {
        "finalizers": [
        ]
    },
    "status": {
        "phase": "Terminating"
    }
}
```

## Step 2: Executing our cleanup command

Now that we have that setup we can instruct our cluster to get rid of that annoying namespace:

```shell
kubectl replace --raw "/api/v1/namespaces/logging/finalize" -f ./logging.json
```

Where: `/api/v1/namespaces/<your_namespace_here>/finalize`

After running that command, the namespace should now be absent from your namespaces list.

The key thing to note here is the resource you are modifying, in our case, it is for namespaces, it could be pods, deployments, services, etc. This same method can be applied to those resources stuck in `Terminating` state.