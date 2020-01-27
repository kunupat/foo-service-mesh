Service mesh resources for `foo` service.


OC Pipelines

Create Tasks

1.	Maven Task-
maven-build-task.yaml

apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: maven-build
spec:
  inputs:
    resources:
    - name: workspace-git
      targetPath: /
      type: git
  steps:
  - name: build
    image: maven:3.6.0-jdk-8-slim
    command:
    - /usr/bin/mvn
    args:
    - install

oc create -f maven-build-task.yaml

2.	Install the s2i-java and openshift-client reusable Tasks from the catalog repository, which contains a list of reusable Tasks for Pipeline s:

$ oc create -f https://raw.githubusercontent.com/openshift/tektoncd-catalog/release-v0.7/openshift-client/openshift-client-task.yaml
$ oc create -f https://raw.githubusercontent.com/openshift/pipelines-catalog/release-v0.7/s2i-java-8/s2i-java-8-task.yaml

3.	Verify the Tasks added to the Pipeline as follows:
$ tkn task ls
 
Create Resources

1.	FooGitPipelineResourece.yaml
2.	apiVersion: tekton.dev/v1alpha1
3.	kind: PipelineResource
4.	metadata:
5.	  name: foo-git
6.	spec:
7.	  type: git
8.	  params:
9.	  - name: url
10.	    value: https://github.com/kunupat/foo

2. FooImageRegistryPipelineResource.yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: foo-image
spec:
  type: image
  params:
  - name: url
    value: image-registry.openshift-image-registry.svc:5000/servicemeshdemo/foo


3.	Create resources
$ oc create -f FooGitPipelineResourece.yaml
$ oc create -f FooImageRegistryPipelineResource.yaml

4.	Create k8s resources- Deployment.yaml, Route.yaml and Service.yaml
Add file snippets here

5.	Create Istio Service Mesh Member Roll
Add file snippet here and PATCH command:
$ oc -n istio-system patch --type='json' smmr default -p '[{"op": "add", "path": "/spec/members", "value":["'"pipelines-demo"'","'"bookinfo"'","'"servicemesh-app"'","'"foo-service"'","'"servicemeshdemo"'"]}]'

6.	Create tasks for applying k8s resources
$ oc create –f Apply_manifests.yaml

7.	Verify resources
	$ tkn resource ls
NAME        TYPE    DETAILS
foo-git     git     url: https://github.com/kunupat/foo
foo-image   image   url: image-registry.openshift-image-registry.svc:5000/servicemeshdemo/foo

Create Pipeline

1.	FooPipeline.yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: foo-deploy-pipeline
spec:
  resources:
  - name: app-git
    type: git
  - name: app-image
    type: image
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
    params:
    - name: ARGS
      value: "rollout latest foo app"

2.	$ oc create -f FooPipeline.yaml
3.	Verify pipeline
$ tkn pipeline ls
NAME                  AGE              LAST RUN                     STARTED          DURATION   STATUS
foo-deploy-pipeline   21 minutes ago   foo-deploy-pipeline-qf5dqk   -2 minutes ago   ---        Running


Run Pipeline

$ tkn pipeline start foo-deploy-pipeline -r app-git=foo-git -r app-image=foo-image -s pipeline



Setting up K8S Resources
Deployment (in project ns)
Service (in project ns)
Setting up Istio Resources
Virtual Service (in project ns)
Gateway (in project ns)
Getting the Istio ingress URL For foo service

export GATEWAY_URL=$(./oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')

