# Creating Openshift CI/CD Pipeline (Tekton Pipeline) with Kubernetes and Istio resources for `foo` service

Follow the below steps to create Tekton Pipeline in OpenShift Service Mesh to deploy [Foo service][1] in Openshift Service Mesh. This pipeline does the following tasks:

1. Pull source code from Git repository & build the Maven project
2. Build Docker image from the source code, label it and push to OpenShift Container Registry
3. Apply the following Kubernetes and Istio Manifests:
	- [DeploymentConfig (OpenShift)][2]
	- [Service (Kubernetes)][3]
	- [Virtual Service (Istio)][4]
	- [Gateway (Istio)][5]

## Reference Links
- Openshift Pipeline Tutorial- https://github.com/openshift/pipelines-tutorial
- Using Pipelines- https://openshift.github.io/pipelines-docs/docs/0.8/assembly_using-pipelines.html

## Creating OpenShift Pipelines

### Create Tasks
Create following tasks:
1. OpenShift Client
2. Java s2i
3. Apply Manifests

#### Create the s2i-java and openshift-client Reusable Tasks

Install the `s2i-java` and `openshift-client` reusable Tasks from the [Tekton Catalog][6], which contains a list of reusable Tasks for Pipelines:

```
$ oc create -f https://raw.githubusercontent.com/openshift/tektoncd-catalog/release-v0.7/openshift-client/openshift-client-task.yaml
$ oc create -f https://raw.githubusercontent.com/openshift/pipelines-catalog/release-v0.7/s2i-java-8/s2i-java-8-task.yaml
```
#### Create ApplyManifests Task
This task runs `oc apply` on the YAMLs present in the `manifests` directory of this repository. The `manifests` directory contains YAML definitions for Kubernetes `Deployment` and `Service` resources and Istio's `Virtual Service` and `Gateway` resources.

*File Name:* `FooService-applyManifestsTask.yaml`

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: apply-foo-manifests
spec:
  inputs:
    resources:
      - {type: git, name: source}
    params:
      - name: manifest_dir
        description: The directory in source that contains yaml manifests
        type: string
        default: "manifests"

  steps:
    - name: apply
      image: quay.io/openshift/origin-cli:latest
      workingDir: /workspace/source
      command: ["/bin/bash", "-c"]
      args:
        - |-
          echo Applying manifests in $(inputs.params.manifest_dir) directory
          oc apply -f $(inputs.params.manifest_dir)
          echo -----------------------------------
```
```
$ oc create -f FooService-applyManifestsTask.yaml
```
Verify that all the Tasks are created correctly:

`$ tkn task ls`

---

### Create Resources
Create Following Resources:
1. Foo Git Pipeline Resource
2. Foo Image Registry Pipeline Resource
3. Foo Service Mesh Git Pipeline Resource

*1. File Name:* `FooGitPipelineResourece.yaml`

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
name: foo-git
spec:
type: gitparams:
	  - name: url
	    value: https://github.com/kunupat/foo
```

`$ oc apply -f FooGitPipelineResourece.yaml`

*2. File Name:*  `FooImageRegistryPipelineResource.yaml`
```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: foo-image
spec:
  type: image
  params:
  - name: url
    value: image-registry.openshift-image-registry.svc:5000/servicemeshdemo/foo
```
`$ oc apply -f FooImageRegistryPipelineResource.yaml`

*3. File Name:* `FooServiceMeshGit-PipelineResourece.yaml`

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: foo-service-mesh-git
spec:
  type: git
  params:
  - name: url
    value: https://github.com/kunupat/foo-service-mesh
```
`$ oc apply -f FooServiceMeshGit-PipelineResourece.yaml`

Verify that the resources are created properly using below command:

```
$ tkn resource ls
```
#### Create Istio Service Mesh Member Roll (Optional/ Required one time only)
In the below file, make sure to replace the value of `/spec/members` path to match to your Openshift environment. Just add your namespace to the existing namespaces listed in `ServiceMeshMemberRoll`.

*File Name:* `ServiceMeshMemberRoll.yaml`

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: apply-foo-manifests
spec:
  inputs:
    resources:
      - {type: git, name: source}
    params:
      - name: manifest_dir
        description: The directory in source that contains yaml manifests
        type: string
        default: "manifests"

  steps:
    - name: apply
      image: quay.io/openshift/origin-cli:latest
      workingDir: /workspace/source
      command: ["/bin/bash", "-c"]
      args:
        - |-
          echo Applying patch to ServiceMeshMemberRoll
          oc -n istio-system patch --type='json' smmr default -p '[{"op": "add", "path": "/spec/members", "value":["'"pipelines-demo"'","'"bookinfo"'","'"servicemesh-app"'","'"foo-service"'","'"servicemeshdemo"'"]}]'
          echo -----------------------------------
```
`$ oc apply -f ServiceMeshMemberRoll.yaml`

---
#### Create Role and RoleBinding for `pipeline` Service Account

We get a similar authorization error as shown below in case `pipeline` Service Account is unauthorized to make API calls on Istio Networking resorces required to apply configuration changes to `Istio Virtual Service` and `Istio Gateway`-
```
User "system:serviceaccount:pipelines-demo:pipeline" cannot get resource "virtualservices" in API group "networking.istio.io" in the namespace...
```
To fix this issue, create a `Role` and `RoleBinding` for `pipeline` Service Account. You can find the respective YAML files for `Role` and `RoleBinding` in /extras directory of this repository. Use following commands to create `Role` and `RoleBinding`:

```
$ oc apply -f /extras/role.yaml
$ oc apply -f /extras/role-binding.yaml
```
Note that this is only required to be done once.

### Create Pipeline

*File Name:* `FooPipelineAll.yaml`

```
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: foo-deploy-pipeline-all-resources
spec:
  resources:
  - name: app-git
    type: git
  - name: app-image
    type: image
  - name: foo-service-mesh-git
    type: git
    
  tasks:
  - name: build
    taskRef:
      name: s2i-java-8
    params:
      - name: TLSVERIFY
        value: "false"
    resources:
      inputs:
      - name: source
        resource: app-git
      outputs:
      - name: image
        resource: app-image
  
  - name: deploy
    taskRef:
      name: openshift-client
    runAfter:
      - build
  
  - name: apply-manifests
    taskRef:
      name: apply-foo-manifests
    resources:
      inputs:
        - name: source
          resource: foo-service-mesh-git
    runAfter:
        - deploy
```

`$ oc create -f FooPipeline.yaml`

Verify pipeline:
`$ tkn pipeline ls`

---

### Run Pipeline
```
$ tkn pipeline start foo-deploy-pipeline -r app-git=foo-git -r app-image=foo-image -s pipeline
```
[1]: https://github.com/kunupat/foo
[2]: https://github.com/kunupat/foo-service-mesh/blob/master/manifests/Deployment-config.yaml
[3]: https://github.com/kunupat/foo-service-mesh/blob/master/manifests/Service.yaml
[4]: https://github.com/kunupat/foo-service-mesh/blob/master/manifests/gateway.yaml
[5]: https://github.com/kunupat/foo-service-mesh/blob/master/manifests/virtual-service.yaml
[6]: https://github.com/tektoncd/catalog
