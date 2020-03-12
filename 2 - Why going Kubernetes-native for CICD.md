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



