# OpenShift Pipelines Tutorial

Welcome to the OpenShift Pipelines tutorial!

OpenShift Pipelines is a cloud-native, continuous integration and delivery (CI/CD) solution for building pipelines using [Tekton](https://tekton.dev). Tekton is a flexible, Kubernetes-native, open-source CI/CD framework that enables automating deployments across multiple platforms (Kubernetes, serverless, VMs, etc) by abstracting away the underlying details.

OpenShift Pipelines features:
  * Standard CI/CD pipeline definition based on Tekton
  * Build images with Kubernetes tools such as S2I, Buildah, Buildpacks, Kaniko, etc
  * Deploy applications to multiple platforms such as Kubernetes, serverless and VMs
  * Easy to extend and integrate with existing tools
  * Scale pipelines on-demand
  * Portable across any Kubernetes platform
  * Designed for microservices and decentralized teams
  * Integrated with the OpenShift Developer Console

This tutorial walks you through pipeline concepts and how to create and run a simple pipeline for building and deploying a containerized app on OpenShift, and in this tutorial, we will use `Triggers` to handle a real GitHub webhook request to kickoff a PipelineRun.

## Prerequisites

You need an OpenShift 4 cluster in order to complete this tutorial. If you don't have an existing cluster, go to http://try.openshift.com and register for free in order to get an OpenShift 4 cluster up and running on AWS within minutes.

You will also use the Tekton CLI (`tkn`) through out this tutorial. Download the Tekton CLI by following [instructions](https://github.com/tektoncd/cli#installing-tkn) available on the CLI GitHub repository.

## Concepts

Tekton defines a number of [Kubernetes custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) as building blocks in order to standardize pipeline concepts and provide a terminology that is consistent across CI/CD solutions. These custom resources are an extension of the Kubernetes API that let users create and interact with these objects using `kubectl` and other Kubernetes tools.

The custom resources needed to define a pipeline are listed below:
* `Task`: a reusable, loosely coupled number of steps that perform a specific task (e.g. building a container image)
* `Pipeline`: the definition of the pipeline and the `Tasks` that it should perform
* `TaskRun`: the execution and result of running an instance of task
* `PipelineRun`: the execution and result of running an instance of pipeline, which includes a number of `TaskRuns`

![Tekton Architecture](docs/images/tekton-architecture.svg)

In short, in order to create a pipeline, one does the following:
* Create custom or install [existing](https://github.com/tektoncd/catalog) reusable `Tasks`
* Create a `Pipeline` and `PipelineResources` to define your application's delivery pipeline
* Create a `PersistentVolumeClaim` to provide the volume/filesystem for pipeline execution or provide a `VolumeClaimTemplate` which creates a `PersistentVolumeClaim`
* Create a `PipelineRun` to instantiate and invoke the pipeline

For further details on pipeline concepts, refer to the [Tekton documentation](https://github.com/tektoncd/pipeline/tree/master/docs#learn-more) that provides an excellent guide for understanding various parameters and attributes available for defining pipelines.

The Tekton API enables functionality to be separated from configuration (e.g.
[Pipelines](https://github.com/tektoncd/pipeline/blob/master/docs/pipelines.md)
vs
[PipelineRuns](https://github.com/tektoncd/pipeline/blob/master/docs/pipelineruns.md))
such that steps can be reusable.

Triggers extend the Tekton
architecture with the following CRDs:

- [`TriggerTemplate`](https://github.com/tektoncd/triggers/blob/master/docs/triggertemplates.md) - Templates resources to be
  created (e.g. Create PipelineResources and PipelineRun that uses them)
- [`TriggerBinding`](https://github.com/tektoncd/triggers/blob/master/docs/triggerbindings.md) - Validates events and extracts
  payload fields
- [`Trigger`](https://github.com/tektoncd/triggers/blob/master/docs/triggers.md) - combines TriggerTemplate, TriggerBindings and interceptors.
- [`EventListener`](https://github.com/tektoncd/triggers/blob/master/docs/eventlisteners.md) -  provides an
  [addressable](https://github.com/knative/eventing/blob/master/docs/spec/interfaces.md)
  endpoint (the event sink). `Trigger` is referenced inside the EventListener Spec. It uses the extracted event parameters from each
  `TriggerBinding` (and any supplied static parameters) to create the resources
  specified in the corresponding `TriggerTemplate`. It also optionally allows an
  external service to pre-process the event payload via the `interceptor` field.
- [`ClusterTriggerBinding`](https://github.com/tektoncd/triggers/blob/master/docs/clustertriggerbindings.md) - A cluster-scoped
  TriggerBinding

Using `tektoncd/triggers` in conjunction with `tektoncd/pipeline` enables you to
easily create full-fledged CI/CD systems where the execution is defined
**entirely** through Kubernetes resources.

You can learn more about `triggers` by checking out the [docs](https://github.com/tektoncd/triggers/blob/master/docs/README.md)

In the following sections, you will go through each of the above steps to define and invoke a pipeline.

## Getting started with Tekton Pipelines, also known as Red Hat OpenShift Pipelines, for people that have never used it before.
This is what the pipeline does:
1.	Fetch the repository
2.	Build the image
3.	Apply the manifests for creating a deployment, a service and a route
4.	Deploy the image

## 1. Deploy pipelines Operator (once per cluster) from the Operator Hub

OpenShift Pipelines is provided as an add-on on top of OpenShift that can be installed via an operator available in the OpenShift OperatorHub. 

* Follow [these instructions](install-operator.md) in order to install OpenShift Pipelines on OpenShift via the OperatorHub.

![OpenShift OperatorHub](docs/images/operatorhub.png)

* Login to the cluster with the provided credentials
* Create your own namespace / project and change to it, this will be where your pipeline will run.
```
oc new-project eric
```
* In order to get familiar with OpenShift Pipelines concepts and create your first pipeline, follow the OpenShift Pipelines Docs but use your own namespace. You can skip the mirroring part.

### 1.1 Create pipeline tasks 

#### Procedure

Install the apply-manifests and update-deployment task resources from the pipelines-tutorial repository, which contains a list of reusable tasks for pipelines:
```
 oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/pipelines-1.18/01_pipeline/01_apply_manifest_task.yaml
 oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/pipelines-1.18/01_pipeline/02_update_deployment_task.yaml
```
Use the tkn task list command to list the tasks you created:
```
 tkn task list
```
The output verifies that the apply-manifests and update-deployment task resources were created:
```
NAME                DESCRIPTION   AGE
apply-manifests                   1 minute ago
update-deployment                 48 seconds ago
```
### 1.2. Assemble a pipeline 

A pipeline represents a CI/CD flow and is defined by the tasks to be executed. It is designed to be generic and reusable in multiple applications and environments.

A pipeline specifies how the tasks interact with each other and their order of execution using the from and runAfter parameters. It uses the workspaces field to specify one or more volumes that each task in the pipeline requires during execution.

In this section, you will create a pipeline that takes the source code of the application from GitHub, and then builds and deploys it on OpenShift Container Platform.

The pipeline performs the following tasks for the application 

Clones the source code of the application from the Git repository by referring to the git-url and git-revision parameters.
Builds the container image using the buildah task provided in the openshift-pipelines namespace.
Pushes the image to the OpenShift image registry by referring to the image parameter.
Deploys the new image on OpenShift Container Platform by using the apply-manifests and update-deployment tasks.

#### Procedure

Copy the contents of the following sample pipeline YAML file and save it:
```
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  workspaces:
  - name: shared-workspace
  params:
  - name: deployment-name
    type: string
    description: name of the deployment to be patched
  - name: git-url
    type: string
    description: url of the git repo for the code of deployment
  - name: git-revision
    type: string
    description: revision to be used from repo of the code for deployment
    default: "main"
  - name: IMAGE
    type: string
    description: image to be built from the code
  tasks:
  - name: fetch-repository
    taskRef:
      resolver: cluster
      params:
      - name: kind
        value: task
      - name: name
        value: git-clone
      - name: namespace
        value: openshift-pipelines
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: URL
      value: $(params.git-url)
    - name: SUBDIRECTORY
      value: ""
    - name: DELETE_EXISTING
      value: "true"
    - name: REVISION
      value: $(params.git-revision)
  - name: build-image
    taskRef:
      resolver: cluster
      params:
      - name: kind
        value: task
      - name: name
        value: buildah
      - name: namespace
        value: openshift-pipelines
    workspaces:
    - name: source
      workspace: shared-workspace
    params:
    - name: IMAGE
      value: $(params.IMAGE)
    runAfter:
    - fetch-repository
  - name: apply-manifests
    taskRef:
      name: apply-manifests
    workspaces:
    - name: source
      workspace: shared-workspace
    runAfter:
    - build-image
  - name: update-deployment
    taskRef:
      name: update-deployment
    params:
    - name: deployment
      value: $(params.deployment-name)
    - name: IMAGE
      value: $(params.IMAGE)
    runAfter:
    - apply-manifests
```
The pipeline definition abstracts away the specifics of the Git source repository and image registries. These details are added as params when a pipeline is triggered and executed.

Create the pipeline:
```
 oc create -f <pipeline-yaml-file-name.yaml>
```
Use the tkn pipeline list command to verify that the pipeline is added to the application:

```
 tkn pipeline list
```
The output verifies that the build-and-deploy pipeline was created:

```
NAME               AGE            LAST RUN   STARTED   DURATION   STATUS
build-and-deploy   1 minute ago   ---        ---       ---        ---
```

## 2. Fork a sample repo nodejs-sample to your own github

For example https://github.com/ericbos111/nodejs-sample

In the git repository, create a folder k8s with 3 files

*	deployment.yaml - copy from vote-ui and edit
*	route.yaml - copy from vote-ui and edit
*	service.yaml - copy from vote-ui and edit

otherwise the apply-manifest task will not find the manifests and the update-deployment task will fail. If your pipeline fails in the update-deployment phase, you will need to edit the manifests. Don’t worry about deploy.yaml and devfile.yaml, they are not referenced.

## 3. Edit Pipeline

In Edit pipeline build-and-deploy, in the yaml file, under DOCKERFILE, put

```
./Dockerfile
```

otherwise the build-image task will not find the Dockerfile. This step is not needed if you used the nodejs-sample repo.

## 4. Start Pipeline

Go to your own namespace / project in the UI.

![Screenshot 2025-05-27 at 13 58 41](https://github.com/user-attachments/assets/6a99ddda-1053-4989-ab95-5b5ddb5fd29a)

#### Parameters

```
deployment-name  * nodejs-sample
git-url          * https://github.com/ericbos111/nodejs-sample
git-revision       main                                                                # or whatever branch you want to pull
IMAGE              image-registry.openshift-image-registry.svc:5000/eric/nodejs-sample # my namespace is 'eric', you will need to use your own.
Timeouts           60 Min
Workspaces
shared-workspace * VolumeClaimTemplate (or existing PersistentVolumeClaim)
```
In the UI, switch to the Developer View, go to Topology, you will see your application there, and in the top right there's a link to your result.

![Screenshot 2025-05-27 at 14 02 50](https://github.com/user-attachments/assets/720c8be3-0d5a-428d-b11d-0aeab4425afc)

Alternatively, in the CLI, you can get the route and then curl the result.
```
oc get route
```
If all tasks succeed but the “Hello from Node.js Starter Application” does not appear in the browser, check if you changed the port to 3001 in the manifests for route and service.

If you want to challenge yourself, edit the application to display a different text in the browser.

## 5. Create a webhook

If your cluster is publicly accessible, you can create a webhook so that the pipeline is triggered every time you commit a change to your application repository.

