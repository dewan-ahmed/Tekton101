Tekton supports source code to be built from GitLab. This short tutorial demonstrates a NodeJS application to be built from GitLab, the image to be pushed in OpenShift internal registry and the application to be deployed on OpenShift.


### Prerequisites

* An OpenShift environment. You can use the [OpenShift Playground](https://www.openshift.com/learn/courses/playground/), but you’ll only have access for 60 minutes (after which time it will be destroyed).
* The [OpenShift Pipelines Operator](https://github.com/openshift/pipelines-tutorial/blob/master/install-operator.md#install-openshift-pipelines) installed on your OpenShift environment. For this tutorial, I used version [0.11.0](https://github.com/openshift/tektoncd-pipeline-operator/tree/v0.11.x).
* The [Tekton command-line interface (CLI)](https://github.com/tektoncd/cli#installing-tkn) tkn. For this tutorial, I used version [0.8.0](https://github.com/tektoncd/cli/tree/v0.8.0#installing-tkn).
* Basic knowledge of [Tekton Pipelines](https://github.com/tektoncd/pipeline).

### Estimated time

Completing the steps in this tutorial should take about 25 to 30 minutes.

### Create a public repository on GitLab 

You can use a respository of your choice but if you'd like to use an existing one, you can fork [this one](https://gitlab.com/dewan_ahmed/catapp).
Follow [GitLab’s documentation](https://docs.gitlab.com/ee/gitlab-basics/fork-project.html) on how to fork a repo to create your own fork of this repo.

### Create a new OpenShift project

This application shows a random cat image so you can name the project **catapp**.

```
oc new-project catapp
```

### Create required Tasks and Pipeline

The following yaml will create two tasks - a **buildah** build task and a **deploy-openshift** task to deploy the application to OpenShift. 

```
# This Task is from the Tekton Catalog (last updated 04/02/18):
# https://github.com/tektoncd/catalog/blob/master/buildah/buildah.yaml
# This Task uses buildah to build and push an image.
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: buildah
spec:
  params:
  - name: BUILDER_IMAGE
    description: The location of the buildah builder image.
    default: quay.io/buildah/stable:v1.11.0
  - name: DOCKERFILE
    description: Path to the Dockerfile to build.
    default: ./Dockerfile
  - name: CONTEXT
    description: Path to the directory to use as context.
    default: .
  - name: TLSVERIFY
    description: Verify the TLS on the registry endpoint (for push/pull to a non-TLS registry)
    default: "true"
  - name: FORMAT
    description: The format of the built container, oci or docker
    default: "oci"

  resources:
    inputs:
    - name: source
      type: git
    outputs:
    - name: image
      type: image

  steps:
  - name: build
    image: $(inputs.params.BUILDER_IMAGE)
    workingDir: /workspace/source
    command: ['buildah', 'bud', '--format=$(inputs.params.FORMAT)', '--tls-verify=$(inputs.params.TLSVERIFY)', '--layers', '-f', '$(inputs.params.DOCKERFILE)', '-t', '$(outputs.resources.image.url)', '$(inputs.params.CONTEXT)']
    volumeMounts:
    - name: varlibcontainers
      mountPath: /var/lib/containers
    securityContext:
      privileged: true

  - name: push
    image: $(inputs.params.BUILDER_IMAGE)
    workingDir: /workspace/source
    command: ['buildah', 'push', '--tls-verify=$(inputs.params.TLSVERIFY)', '$(outputs.resources.image.url)', 'docker://$(outputs.resources.image.url)']
    volumeMounts:
    - name: varlibcontainers
      mountPath: /var/lib/containers
    securityContext:
      privileged: true

  volumes:
  - name: varlibcontainers
    emptyDir: {}
---
# This Task applies Kubernetes resource files with oc apply -f. Then it updates
# the target Deployment image.
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-openshift
spec:
  params:
  - name: K8S_DIRECTORY_PATH
    description: Path to the directory for oc apply -f
    default: config/
  - name: DEPLOYMENT
    description: Name of the Deployment and the container name in the Deployment
  resources:
    inputs:
    - name: source
      type: git
    - name: image
      type: image
  steps:
  - name: apply-config
    image: quay.io/openshift/origin-cli:latest
    workingDir: /workspace/source
    command: ['/bin/bash', '-c']
    args:
    - |-
      oc apply -f $(inputs.params.K8S_DIRECTORY_PATH)
  - name: patch-deployment
    image: quay.io/openshift/origin-cli:latest
    command: ['/bin/bash', '-c']
    args:
    - |-
      oc patch deployment $(inputs.params.DEPLOYMENT) --patch='{"spec":{"template":{"spec":{
        "containers":[{
          "name": "$(inputs.params.DEPLOYMENT)",
          "image":"$(inputs.resources.image.url)"
        }]
      }}}}'
---
# This Pipeline builds and deploys an image on OpenShift.
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-deploy-openshift
spec:
  resources:
  - name: source
    type: git
  - name: image
    type: image
  params:
  - name: DEPLOYMENT
    description: Name of the Deployment and the container name in the Deployment
  - name: K8S_DIRECTORY_PATH
    description: Path to the directory for kubectl apply -f
    default: config/
  tasks:
  - name: build
    taskRef:
      name: buildah
    resources:
      inputs:
      - name: source
        resource: source
      outputs:
      - name: image
        resource: image
    params:
    - name: TLSVERIFY
      value: 'false'
  - name: deploy
    runAfter: [build]
    taskRef:
      name: deploy-openshift
    resources:
      inputs:
      - name: source
        resource: source
      - name: image
        resource: image
    params:
    - name: K8S_DIRECTORY_PATH
      value: $(params.K8S_DIRECTORY_PATH)
    - name: DEPLOYMENT
      value: $(params.DEPLOYMENT)
```

Save this file as _tasks_and_pipeline.yaml_.

### Run the pipeline

Once logged in to your OpenShift cluster, execute the following command:

```
oc apply -f tasks_and_pipeline.yaml
```

This command will create the two tasks and the pipeline. You can list the tasks and the pipeline by following commands:

```
tkn task ls
tkn pipeline ls
```

### Create the PipelineRun

While you have a Pipeline, you'll need a PipelineRun to actually build and deploy the application (think of PipelineRun as an instance of the Pipeline). Execute the following after replacing the _URL_ with your own GitLab repository link:

```
NAMESPACE='catapp'
URL='https://github.com/ncskier/catapp.git' # Replace with your catapp repository url
REVISION='master'
cat << EOF | oc apply -f -
 apiVersion: tekton.dev/v1beta1
 kind: PipelineRun
 metadata:
   name: catapp-build-and-deploy
 spec:
   serviceAccountName: pipeline
   pipelineRef:
     name: build-and-deploy-openshift
   resources:
   - name: source
     resourceSpec:
       type: git
       params:
       - name: revision
         value: ${REVISION}
       - name: url
         value: ${URL}
   - name: image
     resourceSpec:
       type: image
       params:
       - name: url
         value: image-registry.openshift-image-registry.svc:5000/${NAMESPACE}/catapp:${REVISION}
   params:
   - name: DEPLOYMENT
     value: catapp
 EOF
 ```
 
 ### Logs and Monitoring
 
 Watch the PipelineRun:
 
```
tkn pipelinerun list
tkn pipelinerun logs --last -f
```

Verify that CatApp has now built and deployed to your OpenShift environment. CatApp is deployed with an OpenShift Route which exposes the app outside of your cluster. You can get the Route URL using:

```
oc get route catapp --template='http://{{.spec.host}}'
```

![](https://github.com/dewan-ahmed/Tekton101/blob/master/assets/catapp.png)
