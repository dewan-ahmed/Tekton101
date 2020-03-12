Tekton is a powerful yet flexible Kubernetes-native open-source framework for creating continuous integration and delivery (CI/CD) systems. It lets you build, test, and deploy across multiple cloud providers or on-premises systems by abstracting away the underlying implementation details.

## Background

Originated from the [Knative](https://knative.dev/) Build project, Tekton was separated as its own open-source project due to interest and popularity. Since then, Google, IBM, RedHat, Cloudbees and a number of others companies and individual contributors have contributed to this project. Tekton is one of the projects under [Continuous Delivery Foundation](https://cd.foundation/).

## Tekton repositories

[Tekton's GitHub repo](https://github.com/tektoncd) is a combination of various sub-section of this project. The main ones are:

* pipeline
* cli
* dashboard
* triggers
* operator

This article will focus on Tekton Pipelines - the backbone of Tekton project.

## Tekton Pipelines

The Tekton Pipelines project provides Kubernetes-style resources for declaring CI/CD-style pipelines. Tekton Pipelines run on Kubernetes (any Kubernetes), have Kubernetes clusters as a first class type and use containers as their building blocks. 

Tekton Pipelines are Decoupled; meaning one Pipeline can be used to deploy to any Kubernetes cluster. The Tasks which make up a Pipeline can easily be run in isolation as well as the resources such as git repos can easily be swapped between runs.

The concept of typed resources in Tekton Pipelines means that for a resource such as an Image, implementations can easily be swapped out (e.g. building with kaniko v.s. buildkit).

## Tekton Pipelines Building Blocks

Tekton Pipelines uses [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) to create custom resources. Out of the box, Kubernetes comes with resources like pods, deployments and services. But CRDs extend the capability of Kubernetes by letting you define your own resources. Tekton controller acts on these Kubernetes resources.
Here are some of the CRDs that come with Tekton:

* Step: This is an existing type which is Kubernetes container spec. In a Step, you can specify the information on environment variables or volume mounts, etc.
* Task: The first new type is a Task. A Task is a sequence of steps which runs in the order declared and on the same node.
* Pipeline: A Pipeline is made up of Tasks which can run either sequentially, concurrently or as a graph. Tasks within a Pipeline can run on different nodes and you can link output of one Pipeline as an input of another Pipeline.
* TaskRun, PipelineRun & PipelineResource: TaskRun and PipelineRun are instances of Task/Pipeline and fetch runtime configuration from PipelineResource.

The following image will give some clarity on how the above pieces fit together:

![](https://github.com/dewan-ahmed/Tekton101/blob/master/assets/tekton-blocks.png)

Pipeline runs with the privileges of the specified service account. All pods execute a tekton in-built credential initialization step as their first step in each Task. ServiceAccount is patched with Secrets containing the necessary credentials. Tekton currently supports two Kubernetes Secret types: basic-auth and ssh-auth.

In the [next part](https://github.com/dewan-ahmed/Tekton101/blob/master/2%20-%20Why%20going%20Kubernetes-native%20for%20CICD.md) of this series, you'll learn the benefits of Kubernetes-native CI/CD over traditional ones.
