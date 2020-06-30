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
