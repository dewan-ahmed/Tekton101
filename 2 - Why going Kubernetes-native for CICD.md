This is a continuation of a series on Kubernetes-native CI/CD server. To read the first part of the series, visit [An overview of Tekton Pipelines](https://github.com/dewan-ahmed/Tekton101/blob/master/1%20-%20An%20overview%20of%20Tekton%20Pipelines.md).

![](https://github.com/dewan-ahmed/Tekton101/blob/master/assets/too%20many%20tools.png)

Looking at the above image, it is obvious that there is a number of CI/CD tools out there already and new projects are 
being added every month. Its great to have choices but too many choices can lead to confusion and fragmentation. Just like developers, enterprise customers are having challenges making their own tooling decisions. Tekton was born to provide a set of commonly agreed upon Kubernetes-native building blocks for CI/CD systems.

The following table summarizes the challenges around a traditional CI/CD server and compares to a Kubernetes-native CI/CD server:

| Traditional CI/CD Server | Kubernetes-native CI/CD|
|---|---|
| Logging and monitoring is on an external agent| Centralized logging and monitoring|
| High-availability is not possible or complex to achieve| The platform guarantees high-availability|
| Self-healing is not possible (retry logic for some tools)|  Self-healing comes by default –resources are pods|
| Most common CI/CD servers are from a pre-container era| Developed for containers and Kubernetes; runs as containers on Kubernetes|

Let's discuss the above points in depth. For ease of reference, I'll use Jenkins as a traditional CI/CD server example and OpenShift Pipelines as the Kubernetes-native one (that uses Tekton under the hood).

1. **Logging/Monitoring**: In order to view your Jenkins logs, you'd need to access Jenkins server. Although an integration can be made to fetch these logs, this adds overhead, nevertheless. This also means that you'd have someone in your team as the "Jenkins Expert" who understands these logs and can take decisions on it. For OpneShift Pipelines, the resources are pods running in containers. All of these logs are centralized and are shown on the OpenShift Console:

![](https://github.com/dewan-ahmed/Tekton101/blob/master/assets/openshift-pipelines-logs.png)

2. **High-availability**: You can achieve high-availability on Jenkins by using multiple Jenkins masters with HA-proxy. However, you end up using more resource and the process is complex. For OpenShift Pipelines, the cluster itself can offer high-availability if chosen the right minimum number of master nodes.

3. **Self-healing**: While Jenkins can implement _retry_ logic for its failed stages, it is limited to scope and number. Kubernetes (OpenShift, in this case), as a software, has self-healing baked in. This means all the resources run as pods and if one goes down, the platform automatically brings another up to continue the task. 

4. **Pre-container**: Although Jenkins uses a number of plugins to provide cloud-native capabilities, the fact that these plug-ins do not feel native is primarily since Jenkins was developed at a time when containers were not prevalent. 

**Work-in-progress**
