Service mesh resources for `foo` service.


# Creating OpenShift Pipelines

## Create Tasks

### Install the s2i-java and openshift-client Reusable Tasks

Install the s2i-java and openshift-client reusable Tasks from the catalog repository, which contains a list of reusable Tasks for Pipelines:

```
$ oc create -f https://raw.githubusercontent.com/openshift/tektoncd-catalog/release-v0.7/openshift-client/openshift-client-task.yaml
$ oc create -f https://raw.githubusercontent.com/openshift/pipelines-catalog/release-v0.7/s2i-java-8/s2i-java-8-task.yaml
```

### Verify the Tasks added to the Pipeline as follows:
$ tkn task ls
 
## Create Resources

### FooGitPipelineResourece.yaml
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

### FooImageRegistryPipelineResource.yaml
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

### Create resources
```
$ oc create -f FooGitPipelineResourece.yaml
$ oc create -f FooImageRegistryPipelineResource.yaml
```

### Create Istio Service Mesh Member Roll (Optional)
Add file snippet here and PATCH command:
```
$ oc -n istio-system patch --type='json' smmr default -p '[{"op": "add", "path": "/spec/members", "value":["'"pipelines-demo"'","'"bookinfo"'","'"servicemesh-app"'","'"foo-service"'","'"servicemeshdemo"'"]}]'
```

### Verify resources
```
$ tkn resource ls
```

## Create Pipeline

### FooPipelineAll.yaml
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

### Create Pipeline
```
oc create -f FooPipeline.yaml
```

### Verify pipeline
```
$ tkn pipeline ls
```

### Run Pipeline
```
$ tkn pipeline start foo-deploy-pipeline -r app-git=foo-git -r app-image=foo-image -s pipeline
```
