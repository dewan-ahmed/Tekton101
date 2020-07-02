This is a basic Tekton pipelines demo for building an image from github and pushing it to docker hub. Throughout this demo, k8s is used as an alias for Kubernetes.

## Pre-requisites

* A k8s cluster: Any k8s cluster will do. To get a free cluster on IBM Cloud, [follow this link](https://www.ibm.com/cloud/kubernetes-service). This demo will consider an IBM Cloud Kubernetes Service.
* Installation of [Tekton CLI](https://github.com/tektoncd/cli)
* Installation of [latest Tekton release](https://github.com/tektoncd/pipeline/blob/master/docs/install.md#installing-tekton-pipelines)
* A GitHub repo to pull the source code from and a Docker Hub (or any other image registry) to push the image to

## The setup

![](https://github.com/dewan-ahmed/Tekton101/blob/master/assets/arch.png)

## Step1 - Verifying required setup

* `kubectl version --short` to ensure you're targeting a k8s cluster (and to check the version). If you don't see an output or if the command is not recognized, refer to [this link](https://cloud.ibm.com/docs/containers?topic=containers-cs_cli_install#kubectl).
* `tkn` will show list of available commands and this is an indication of a successful Tekton CLI installation.
* Once Tekton CRDs are created, we can verify Tekton Pipelines installation.

## Step2 - Creating k8s secrets

Create the following file and save as _docker-secret.yaml_. Take a note of the _kind_ as _Secret_.
```
apiVersion: v1
kind: Secret
metadata:
  name: basic-user-pass-docker
  annotations:
    tekton.dev/docker-0: https://index.docker.io # Described below
type: kubernetes.io/basic-auth
stringData:
  username: <your_docker_username>
  password: <your_docker_password>
```
Run `kubectl apply -f docker-secret.yaml` to create the necessary secret. Similar secret for GitHub is not required as we're using a public repository.

## Step3 - Creating a service account

Create the following file and save as _serviceaccount.yaml_. Take a note of the _kind_ as _ServiceAccount_ and you're referring previously created _secret_ under the _metadata_.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-sa
secrets:
  - name: basic-user-pass-docker
```

Run `kubectl apply -f serviceaccount.yaml` to create the necessary service account.

## Step3 - Creating necessary pipeline resources

The followings are _PipelineResource_ files for git and docker. Copy the first block of code and save as _git-resource.yaml_ and the seconds block of code as _docker-resource.yaml_.

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git-source
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: https://github.com/dewandemo/simple-nodejs-app
```
Feel free to use your own public github repository provided that it has a dockerfile. For private repository, you'll need to create a github secret file similar to the docker secret we created in Step2.

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: docker-target
spec:
  type: image
  params:
    - name: url
      value: index.docker.io/<your-docker-username>/<your-docker-reponame>
```

Once you create these files, run `kubectl apply -f git-resource.yaml` and `kubectl apply -f docker-resource.yaml` to create the necessary PipelineResource.

## Step4 - Creating the Task

This step creates the Tekton CRD - Task. Kaniko is used as the build tool. Create a new file with following contents and save as _Task.yaml_.

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-docker-image-from-git-source
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToDockerFile
        type: string
        description: The path to the dockerfile to build
        default: /workspace/docker-source/Dockerfile
      - name: pathToContext
        type: string
        description:
          The build context used by Kaniko
          (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
        default: /workspace/docker-source
  outputs:
    resources:
      - name: builtImage
        type: image
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:v0.15.0
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(outputs.resources.builtImage.url)
        - --context=$(inputs.params.pathToContext)
```
If you're using a different git source, observe *default: /workspace/docker-source* which might change. Run `kubectl apply -f Task.yaml` to create the _Task_. The Task is not running yet as a _TaskRun_ needs to "run" on the _Task_.

## Step5 - Creating the TaskRun

Create a new file with the following content and save as _TaskRun.yaml_:

```
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: build-docker-image-from-git-source-task-run
spec:
  serviceAccountName: tekton-sa
  taskRef:
    name: build-docker-image-from-git-source
  inputs:
    resources:
      - name: git-source
        resourceRef:
          name: git-source
    params:
      - name: pathToDockerFile
        value: Dockerfile
      - name: pathToContext
        value: /workspace/git-source
  outputs:
    resources:
      - name: builtImage
        resourceRef:
          name: docker-target
```

Notice how the _TaskRun_ brings all the pieces together. Under _spec_, it refers to the _serviceaccount_ we created earlier, the _taskRef_ with name of the _Task_ and _git-source_ and _docker-target_ under _inputs_ and _outputs_.

Run `kubectl apply -f TaskRun.yaml` to create the _TaskRun_.

## Step6 - Logs and observation

Run `tkn taskrun describe build-docker-image-from-git-source-task-run` to see the status of the _TaskRun_. You can also run `tkn taskrun logs build-docker-image-from-git-source-task-run` to watch the detailed logs. If the steps were successful, you should be able to see an image pushed to your docker hub repo shortly. Congratulations on finishing this lab on Tekton!
