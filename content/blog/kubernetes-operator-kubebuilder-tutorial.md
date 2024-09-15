+++
title = 'Build Your First Kubernetes Operator'
draft = false
+++


## What is a Kubernetes Operator and Why Build One From Scratch?
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Kubernetes Operators enable teams to automate the operation of Kubernetes resources and applications. For complex projects with predictable patterns of maintenance and operation, Operators can be a powerful tool for simplifying the management of applications running on Kubernetes.<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Unlike standard Helm charts, Operators are not limited to configurations that can only be described using languages like YAML. Instead, you can use powerful programming languages like Golang and Java to automate a plethora of tasks within your clusters. This makes Operators a great option when the standard functionality provided by tools like Helm and Kustomize have been exhausted and a more robust solution is needed.<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;If you want to learn more about the underlying concepts behind popular tools like Rook and Crossplane, then following this tutorial will help you do just that by walking you through the process of building a Kubernetes Operator from scratch.

## Overview

### Tutorial Structure
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The goal of this tutorial is to help learners get a custom Kubernetes Operator up and running as quickly as possible. In order to accomplish this, many details explaining the <I>why</I> of the Operator are left out. This approach might seem counter intuitive but for some, myself included, this is an important first step in learning a new technology.<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Additionally, we will basically speed run through the [official Kubebuilder tutorial](https://www.kubebuilder.io/cronjob-tutorial/cronjob-tutorial) which can be referenced at any time to gain a deeper understanding of what's going on.
### What We're Building
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Finally, the Operator we will build supports a custom CronJob resource that will make using CronJobs more user friendly. The role of a CronJob in Kubernetes is to create scheduled containerized Jobs that complete one-off tasks.

## Prerequisites

Once you have installed the tools listed below, we are ready to get started. This tutorial assumes only a Beginner to Intermediate understanding of these tools.

<ul>
<li><b><a href="https://go.dev">Golang:</a></b> The programming language we will use to build the Operator</li>
<li><b><a href="https://www.kubebuilder.io/quick-start">kubebuilder:</a></b> The framework we will use to simplify the development of a Kubernetes Operator
<li><b><a href="https://kubernetes.io/docs/tasks/tools/#kubectl">kubectl:</a></b> A command line tool used to communicate with Kubernetes clusters</li>
<li><b><a href="https://www.docker.com">Docker:</a></b> The container runtime our Kubernetes cluster will use</li>
<li><b><a href="https://kind.sigs.k8s.io/docs/user/quick-start#installing-with-a-package-manager">kind:</a></b> A Docker based tool used to spin up local Kubernetes clusters</li> 
</ul>

## Step 1: Boilerplate

The first step to building the operator is to initialize a new Kubebuilder project.
```
mkdir project
cd project
kubebuilder init --domain tutorial.kubebuilder.io --repo tutorial.kubebuilder.io/project
```

## Step 2: Define Custom Resource APIs
Next, we will create a new API that represents our custom CronJob resource. When prompted, press y for "Create Resource" and "Create Controller".
```
kubebuilder create api --group batch --version v1 --kind CronJob

```
Next, overwrite the contents `project/api/v1/cronjob_types.go` with the code shared here: [project/api/v1/cronjob_types.go @ github.com/kubernetes-sigs/kubebuilder](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/api/v1/cronjob_types.go)

Key changes include describing what fields our custom resource will have (CronJobSpec) and what states the resource can be in (CronJobStatus).
## Step 3: Implement a Controller for the Custom Resource
Right now, if you fill out a template (manifest) to deploy the custom CronJob resource, it won't really do anything. That's because custom resource definitions need a controller to tell them what to do (<I>"control"</i> them). This is similar to how an Ingress resource won't work until a compatible Ingress Controller is deployed.

Like before, overwrite the contents of `project/internal/controller/cronjob_controller.go` with the code shared here: [project/internal/controller/cronjob_controller.go @ github.com/kubernetes-sigs/kubebuilder](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/internal/controller/cronjob_controller.go)

Key changes made include filling out the `Reconcile` and `SetupWithManager` function.

### The Reconcile function
The Reconcile function will read the current state of the Kubernetes cluster, then perform actions to match desired state described by the custom resources. In essence, it <i>"reconciles"</i> the current state of the cluster to the described target state. Throughout this function, the `CronJobReconciler` (variable `r`) is used to `Get`, `List`, `Update`, `Delete`, and `Create` resources to realize the target state.

### The SetupWithManager function
Similar to how the kube-controller-manager manages controllers that are native to Kubernetes clusters, we need to register the newly created controller with its own controller manager. In general, a Manager is an executable that wraps one or more Controllers.

## Step 4: Run and Deploy the Controller and Custom Resources

### Create a Kind Kubernetes cluster

At this point we're finally read to get everything running on a Kubernetes cluster. To bootstrap a new local Kind cluster, create a file called `cluster-config.yaml` with the contents shown below.

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: kind
nodes:
  - role: control-plane
    image: kindest/node:v1.30.0@sha256:047357ac0cfea04663786a612ba1eaba9702bef25227a794b52890dd8bcd692e
  - role: worker
    image: kindest/node:v1.30.0@sha256:047357ac0cfea04663786a612ba1eaba9702bef25227a794b52890dd8bcd692e
  - role: worker
    image: kindest/node:v1.30.0@sha256:047357ac0cfea04663786a612ba1eaba9702bef25227a794b52890dd8bcd692e
```
Then run the command `kind create cluster --config cluster-config.yaml`.

Confirm you are able to communicate with the newly created cluster by running `kubectl config current-context`. This command should output `kind-kind`. If not, update the kubectl context by running `kubectl use-context kind-kind`.

### Generate and Install Custom Resource Manifests

Next we'll generate and run the custom resource manifests. A Makefile is used to run the commands. Most of the commands use `kubectl apply` which has limits that may prevent `make install` from running successfully. To resolve this, we will replace the following line in the Makefile.

<b>Before</b>
```
$(CONTROLLER_GEN) rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
```
<b>After</b>
```
$(CONTROLLER_GEN) rbac:roleName=manager-role crd:maxDescLen=0 webhook paths="./..." output:crd:artifacts:config=config/crd/bases
```
After this, you can run these commands to generate and install the manifests:

```
go mod tidy 
make manifests
make install
```

### Test the Controller Locally
We will now test the controller locally before finally deploying it to the cluster. In another terminal, run the following commands within the `project` folder.
```
export ENABLE_WEBHOOKS=false
make run
```

We can then replace the contents of `config/samples/batch_v1_cronjob.yaml` with the following manifest:
```
apiVersion: batch.tutorial.kubebuilder.io/v1
kind: CronJob
metadata:
  labels:
    app.kubernetes.io/name: project
    app.kubernetes.io/managed-by: kustomize
  name: cronjob-sample
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 60
  concurrencyPolicy: Allow # explicitly specify, but Allow is also default.
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
  

```
Then we'll deploy the CronJob to the cluster and confirm it is created successfully.
```
kubectl create -f config/samples/batch_v1_cronjob.yaml
kubectl get cronjob.batch.tutorial.kubebuilder.io -o yaml
```
If you now check the terminal that is running `make run`, you will see many new logs that indicate that the controller is working as expected. For example, after a few minutes you'll see the value of `"successful jobs"` increases by 1 every minute which aligns with the configuration of our CronJob. You will also see the successfully completed jobs when you run `kubectl get jobs`.

### Deploy the Controller to the Kind Cluster
Now that we have confirmed the controller is working as expected, we can deploy it to the cluster.

But first, let's stop the locally running controller by pressing `Ctrl + C` in the terminal. You'll notice that after doing so, you won't see any newly completed jobs when running `kubectl get job`. This again goes to show that without the controller, our Custom Resources won't do much.

Run these commands to deploy the controller to the kind cluster.
```
make docker-build IMG=kubebuilder-tutorial/cronjob-controller:0.0.1
kind load docker-image kubebuilder-tutorial/cronjob-controller:0.0.1
make deploy IMG=kubebuilder-tutorial/cronjob-controller:0.0.1
```
We can confirm the controller was deployed successfully by running:
```
kubectl get pods,services -n project-system                                   
```
Additionally, you'll see that new jobs are being created once again:
```
kubectl get jobs
```
## Step 5: Testing the Operator
### Write and Run Unit Tests
The functionality of the controller should be tested thoroughly to prevent unexpected behaviour. By default, two test files for the controller is included in `project/internal/controller/`. You can run the tests by running
```
make test
```
Currently, the test will fail but if you replace `project/internal/controller/cronjob_controller_test.go` with [project/internal/controller/cronjob_controller_test.go @ github.com/kubernetes-sigs/kubebuilder](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/internal/controller/cronjob_controller_test.go) and `project/internal/controller/suite_test.go` with [project/internal/controller/suite_test.go @ github.com/kubernetes-sigs/kubebuilder](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/internal/controller/suite_test.go), they should pass. Note, you may need to 

### Run End-to-End Tests
You can implement automated end-to-end tests that mimic real user behaviour using Kind. The default kubebuilder boilerplate includes a simple end-to-end test in the folder `project/test/e2e/` that you can run. It requires that an existing Kind cluster named `kind` is running.
```
make test-e2e
```
## Step 6 (optional): Implement WebHooks

Last but not least, we will cover how to create HTTP admission webhooks for the Operator. First we will create boilerplate for the new webhook by running:
```
kubebuilder create webhook --group batch --version v1 --kind CronJob --defaulting --programmatic-validation
```
The above command specifically scaffolds a "defaulting" and "validating" web hook. The "Defaulter" will set defaults for the custom resource. The "Validator" will further validate the custom resource beyond what's possible through the YAML manifests alone.

After this, replace `project/api/v1/cronjob_webhook.go` with [project/api/v1/cronjob_webhook.go @ github.com/kubernetes-sigs/kubebuilder](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/api/v1/cronjob_webhook.go).

HTTPS encrypted communication is required to run the webhook. We will retrieve free SSL certs by using cert-manager:
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
```
We will then need to rebuild the controller with a different image tag than before.
```
make docker-build IMG=kubebuilder-tutorial/cronjob-controller:0.0.2
kind load docker-image kubebuilder-tutorial/cronjob-controller:0.0.2
``` 
Then we need to uncomment all Kustomize paths, resources, and replacements labelled `[WEBHOOK]` or `[CERTMANAGER]` in `project/config/crd/kustomization.yaml` and `project/config/default/kustomization.yaml`.

Finally, we can re-deploy the controller with the new image:
```
make deploy IMG=kubebuilder-tutorial/cronjob-controller:0.0.2
```
Now if you try to create a second CronJob with an invalid schedule, the Validator should catch the issue and return an error:
```
apiVersion: batch.tutorial.kubebuilder.io/v1
kind: CronJob
metadata:
  labels:
    app.kubernetes.io/name: project
    app.kubernetes.io/managed-by: kustomize
  name: invalid-cronjob-sample
spec:
  schedule: "*/1 * * * * * *"
  startingDeadlineSeconds: 60
  concurrencyPolicy: Allow # explicitly specify, but Allow is also default.
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
  
```
## Conclusion
And with that, we have covered the entire Kubebuilder tutorial in a condensed manner. You can delete the cluster you created by running `kind delete cluster -n kind`. When you're ready, I encourage you to go back to the [official Kubebuilder tutorial](https://www.kubebuilder.io/cronjob-tutorial/cronjob-tutorial) and read the explanations more thoroughly to get a deeper understanding of building Operators with Kubebuilder. Thank you for reading!
