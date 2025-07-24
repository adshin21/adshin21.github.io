---
layout: post
title: 'Running Kubernetes Jobs with Go: A Practical Guide with Minikube and Client-go'
description: >-
   Learn how to create and run Kubernetes Jobs using Go, Minikube, and the client-go library with a hands-on example.
date: 2025-07-24 21:27 +0530
categories: [Blogging, Tutorial]
tags: [kubernetes, golang, client-go, job]
---


In this post, we'll walk through how to create [Kubernetes](https://kubernetes.io/) Jobs using Go. We'll be understanding the basic concept of Kubernetes required for running the job locally with [Minikube](https://minikube.sigs.k8s.io/docs/). So let's begin.

## Kubernetes

> [Kubernetes](https://kubernetes.io/docs/concepts/overview/), also known as K8s, is an open source system for automating deployment, scaling, and management of containerized applications.

 **Kubernetes** is like a super-smart manager for your apps, making sure they run smoothly, scale up or down as needed, and deploy new versions without a hitch. It basically automates all the complex stuff of managing many applications, freeing you up to focus on building awesome software. Think of it as your reliable assistant for handling apps on a large scale.

Kubernetes itself is a broader concept and will require an ample amount of time to understand and master. Their documentation can be found [here](https://kubernetes.io/docs/home/). Although, we'll be focusing on a few components required for this post.

There are different controllers, like [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/), HPA, etc., that run or manage pods.

## Pods

> [_Pods_](https://kubernetes.io/docs/concepts/workloads/pods/) are the smallest deployable units of computing that you can create and manage in Kubernetes.

Inside a pod, one or more containers (like Docker containers) share the same network and storage and easily talk to each other. So while a container is the app itself, a pod is how Kubernetes manages those apps into a single group.

Sample YAML example of a Pod of running [`Nginx`](https://nginx.org/)

`my-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-simple-web-server-pod
spec:
  containers:
  - name: simple-nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

## Kubernetes Job

> Jobs represent one-off tasks that run to completion and then stop.

A job creates one or more pods depending upon the job spec and runs it till the completion criteria are met or failure. Once the job finishes, it will clean up its own resources. No additional steps should be required if it is configured properly.

Jobs are great for batch tasks, scripts, or one-off tasks. It contrasts with Deployment, which runs continuously.

Simple YAML example of a Job that will run the echo command. We'll be running the same job from our go code.

`my-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo", "Hello from Kubernetes job!"]
      restartPolicy: Never
```

## Kubectl

> Kubernetes provides a command line tool for communicating with a Kubernetes cluster's [control plane](https://kubernetes.io/docs/reference/glossary/?all=true#term-control-plane), using the Kubernetes API.

[Kubectl](https://kubernetes.io/docs/reference/kubectl/) is a CLI tool to manage and interact with Kubernetes clusters. If anyone has worked with Kubernetes, it is a well-known tool, although we'll not be using it in this post, but it can help you debug and can save you time.

The above YAML file can be used with it to run the defined pod and job.

```bash
kubectl apply -f my-pod.yaml

kubectl apply -f my-job.yaml
```

The basic command and its usage can be found [here](https://kubernetes.io/docs/reference/kubectl/#examples-common-operations).
Since we've got enough understanding of Kubernetes and its components required to proceed further. Now we'll be setting up to run it local before diving into Go code.

## Minikube Setup

There are multiple ways you can run Kubernetes , such as [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download), [Kind](https://kind.sigs.k8s.io/), and [K3s](https://k3s.io/).
We're going to use Minikube with Docker. (Since I'm personally using it 😅).

Use the official doc to install as per your system architecture.

>To install the latest minikube **stable** release on **ARM64** **macOS** using **binary download**:

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
```

After installation, let's start it by

```bash
minikube start
```

 It'll take some time to pull the required images and do the initial setup.

Check the status by running

```bash
minikube status
```

The output should be

```bash
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
docker-env: in-use
```

Since the setup is done, we can use `kubectl` commands to run to pods or job.

---

## Implementation

We'll be creating a simple Go application that'll be runnable on local.

### Project Setup

We'll be using the [`client-go`](https://github.com/kubernetes/client-go) to interact with Kubernetes.

Create a directory and initialise the Go package.

```
mkdir k8s-job-demo

cd k8s-job-demo

go mod init github.com/yourname/k8s-job-demo
```

### Structure

#### File Tree

```
~/k8s-job-demo/
├───go.mod
├───go.sum
├───main.go
└───k8sutil/
    ├───client.go
    ├───job-spec.go
    └───run-job.go
```

- `main.go` will be the entry point.
- All the interaction with k8s will be present inside the package `k8sutil`.

### Connecting with Kubernetes

We'll be connecting our Go application to Kubernetes using the default config, but our function accepts a config file.

`k8sutil/client.go`

```go
// Package k8sutil will implement utility to operate k8s
package k8sutil

import (
 "fmt"
 "path/filepath"

 "k8s.io/client-go/kubernetes"
 "k8s.io/client-go/rest"
 "k8s.io/client-go/tools/clientcmd"
 "k8s.io/client-go/util/homedir"
)

// newClient returns a kubernetes clientset from the given config path
// if path is empty it'll try to load the default kube config else error
func newClientSet(kubeconfig string) (*kubernetes.Clientset, error) {
 var config *rest.Config
 var err error

 // try loading from default config
 if len(kubeconfig) == 0 {
  if home := homedir.HomeDir(); home != "" {
   kubeconfig = filepath.Join(home, ".kube", "config")
  } else {
   return nil, fmt.Errorf("could not find kubeconfig")
  }
 }
 config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
 if err != nil {
  return nil, fmt.Errorf("error loading given file: %w", err)
 }

 return kubernetes.NewForConfig(config)
}
```

### Creating Job Specification

This is similar to the above job YAML, we're representing the `my-job.yaml` to its respective Go struct.

`k8sutil/job-spec.go`

```go
package k8sutil

import (
 "fmt"
 "time"

 batchv1 "k8s.io/api/batch/v1"
 corev1 "k8s.io/api/core/v1"

 metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// createJobSpec returns a job object that can be applied to cluster
// It'll return the yaml example to k8s job object
func createJobSpec(name string) *batchv1.Job {
 // add current timestamp, as job name should be unique
 name = fmt.Sprintf("%s--%d", name, time.Now().UTC().UnixMilli())
 return &batchv1.Job{
  ObjectMeta: metav1.ObjectMeta{
   Name: name,
  },
  Spec: batchv1.JobSpec{
   Template: corev1.PodTemplateSpec{
    Spec: corev1.PodSpec{
     Containers: []corev1.Container{
      {
       Name:    name,
       Image:   "busybox",
       Command: []string{"sh", "-c", "echo Hello from Kubernetes Job! && sleep 30"},
      },
     },
     RestartPolicy: corev1.RestartPolicyNever,
    },
   },
  },
 }
}
```

### Run Job

This function will be called from `main.go` and eventually trigger the job.

- We'll initialise the clientset.
- Then get the job spec.
- Now using the job client, we'll trigger the job.

`k8sutil/job-spec.go`

```go
package k8sutil

import (
 "context"
 "fmt"
 "log/slog"

 metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// RunSampleJob runs a simple job assuming default config
func RunSampleJob() error {
 // get kubernetes clientset
 clientset, err := newClientSet("")
 if err != nil {
  return fmt.Errorf("error getting k8s clientset: %w", err)
 }

 // get job spec
 job := createJobSpec("hello")

 // create a client for default namespace
 jobClient := clientset.BatchV1().Jobs("default")

 slog.Info("Triggering a sample job", "name", job.Name)
 // trigger the job
 _, err = jobClient.Create(context.TODO(), job, metav1.CreateOptions{})
 if err != nil {
  return fmt.Errorf("error creating job: %w", err)
 }

 slog.Info("Job has been created successfully", "name", job.Name)
 return nil
}
```

### main.go

Calling the run job function

```go
package main

import (
 "log/slog"
 "os"

 "github.com/adshin21/k8s-job-demo/k8sutil"
)

func main() {
 if err := k8sutil.RunSampleJob(); err != nil {
  slog.Error("error triggering the job", "err", err)
  os.Exit(1)
 }
}
```

## Execution

Now our setup is ready to run it

```bash

# it'll install all the dependencies
go mod tidy 

# then
go run main.go
```

After running you should see the output; the job has been triggered.

```bash
2025/07/24 17:55:58 INFO Triggering a sample job name=hello--1753359958814
2025/07/24 17:55:58 INFO Job has been created successfully name=hello--1753359958814
```

Now this job will run for 30 seconds since we've added a sleep of 30 seconds.

```bash
kubectl get jobs
# output
# hello--1753359958814   Complete   1/1           36s        99s

kubectl get pods
# output
# hello--1753359958814-p4j4s   0/1     Completed   0          97s

# You can also see logs

kubectl logs jobs/hello--1753359958814
# output
# Hello from Kubernetes Job!
```

🥳 You've successfully executed the job.

## Cleanup

Did you remember? We've mentioned above that
>Once the job finishes, it will clean up its own resources.

You might notice something interesting: if you run that Go program multiple times, the list of **jobs** and **pods** in your Kubernetes cluster will keep growing. This happens because, by default, a Job cleans up its _resources_ (like the running pod) once it's done, but it doesn't remove its _metadata_ —basically, the record of that job existing.

Now, there's a simple fix for this! In your Job's specification (**Job Spec**), you can set a flag called `TTLSecondsAfterFinished`. Once you set this, Kubernetes will automatically clean up _all_ the metadata related to that Job after the time interval you specify. This means everything will be properly removed!

I'd recommend you try implementing this yourself and see how it works. After this cleanup, you'll even be able to reuse the same job name again (though generally, that's not advisable for good practice).

Just a quick heads-up: always make sure you've stored any important logs or necessary information elsewhere before you set up this automatic cleanup!

## Conclusion

The above implementation can be found on [GitHub Repo](https://github.com/adshin21/k8s-job-demo/tree/main).

This was a very basic demo, and the idea behind it was to have an understanding interaction of K8S and Golang.

There are multiple things I've not covered, and there are things like RBAC, logging, debugging and tracing to be implemented to run it in production.

The **`client-go` SDK** is seriously well-made! When I first started using it, I hardly even needed to check the official documentation because everything was explained so clearly right there in the code itself. If you're new to the **Go programming language**, you should definitely take some time to explore it. It's a fantastic example of well-written and documented code.

PS: I've taken some references from the official documentation and even used their example.

---
This was my first write-up. Feel free to write me for suggestions, queries, improvements, and anything else on [LinkedIn](https://www.linkedin.com/in/adshin21/).
