Tekton supports source code to be built from GitLab. This short tutorial demonstrates a NodeJS application to be built from GitLab and the image to be pushed in Docker hub.

### Prerequisites

* Basic understanding of Git, Kubernetes, Tekton and it's components.
* A Kubernetes cluster: Any Kubernetes cluster will do. To get a free cluster on IBM Cloud, [follow this link](https://www.ibm.com/cloud/kubernetes-service).
* Installation of [Tekton CLI](https://github.com/tektoncd/cli) on your machine.
* Installation of [latest Tekton Pipelines release](https://github.com/tektoncd/pipeline/blob/master/docs/install.md#installing-tekton-pipelines).
* A public GitLab repo to pull the source code from (without the need to authenticate) and a Docker Hub (or any other image registry) to push the image to.

### Create Kubernetes Secret and ServiceAccount

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
Run `kubectl apply -f docker-secret.yaml` to create the necessary secret. Similar secret for GitLab is not required as we're using a public repository.

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

## Create pipeline resources

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
      value: https://gitlab.com/dewan_ahmed/simple-nodejs-app
```
Feel free to use your own public GitLab repository provided that it has a Dockerfile. For private repository, you'll need to create a GitLab secret file similar to the docker secret we created. If using a Personal Access Token for _password_ field, you can use any value for _username_ field as this will be ignored.

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
