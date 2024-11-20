# Tekton builds of pre-req images

Used to build minimal ACE images that can then be used to build and run ACE applications.

![Image build overview](ace-demo-pipeline-tekton-2.png)

These images are not required for the successful use of the demo pipeline, and 
others can be used for build and test:

- The `ace` image from cp.icr.io can be used as a build image and also as a runtime 
  image for the various containers.
- The `ace-server-prod` image can be used as a runtime image for CP4i users.

See [ACE containers: choosing a base image](https://community.ibm.com/community/user/integration/blogs/trevor-dolby/2024/02/05/ace-containers-choosing-a-base-image)
for a discussion on how to decide along with some of the history of the images.

The minimal images can be helpful in some cases:

- For users who do not have an IBM Entitlement Key and therefore cannot use the 
  `ace` image, `ace-minimal` can be built from the freely-available ACE Developer
  edition and used to experiment with ACE and pipelines.
- Maven is not installed in the `ace` image and so users wishing to run Maven 
  builds will need to create a new image with Maven installed. Note that Maven is
  no longer required for the demo pipeline to successfully run. 
- In some cases, the container builds using buildah or Kaniko are unable to cache
  container images locally, leading to delays in unpacking the images every time.
  For these situations, `ace-minimal` is faster due to the small image size.

## Getting started

Many of the artifacts in this repo (such as ace-minimal-build-image-pipeline-run.yaml) will need to be 
customized depending on the exact cluster layout. The defaults are set up for Minikube running with Docker
on Ubuntu, and may need to be modified depending on network addresses, etc. The most-commonly-modified 
files have options in the comments, with [ace-minimal-build-image-pipeline-run.yaml](ace-minimal-build-image-pipeline-run.yaml)
being one example:
```
    - name: dockerRegistry
      # OpenShift
      #value: "image-registry.openshift-image-registry.svc.cluster.local:5000/default"
      #value: "quay.io/trevor_dolby"
      #value: "us.icr.io/ace-containers"
      #value: "aceDemoRegistry.azurecr.io"
      # Minikube
      value: "192.168.49.2:5000/default"
```

The Tekton pipeline and ACE runtime rely on having permission to access the container registry,
and this may require the provision of credentials for the service accounts to use:

- Minikube container registry does not have authentication enabled by default, and so dummy
credentials can be used for the `regcred` secret:
  ```
  kubectl create secret docker-registry regcred --docker-server=us.icr.io --docker-username=notused --docker-password=notused
  kubectl apply -f tekton/service-account.yaml
  ```
- OpenShift container registry does have authentication enabled, but this is integrated and requires
only that the service accounts have the `system:image-builder` role and dummy credentials can be used:
  ```
  kubectl create secret docker-registry regcred --docker-server=us.icr.io --docker-username=notused --docker-password=notused
  kubectl apply -f tekton/service-account.yaml
  ```
  It is also possible to use the logged-in user's token, which may be necessary in some cases:
  ```
  kubectl create secret docker-registry regcred --docker-server=image-registry.openshift-image-registry.svc.cluster.local:5000 --docker-username=kubeadmin --docker-password=$(oc whoami -t)
  ```
  See the "Openshift" section below for more details.
- External registries normally require authentication, and in that case the default runtime 
service account needs to be given the credentials:
  ```
  kubectl create secret docker-registry regcred --docker-server=us.icr.io --docker-username=<user> --docker-password=<password>
  kubectl patch serviceaccount default --type=json -p='[{"op": "add", "path": "/imagePullSecrets/-", "value": {"name": "regcred"}}]'
  kubectl apply -f tekton/service-account.yaml
  ```
The service account also has the ability to create services, deployments, etc, which are necessary for running the service. 
Note that on Windows, kubectl sometimes complains about not being able to validate files (using --validate=false appears to 
eliminate the issue without causing problems) and seems to need the `--docker-email` parameter also, but the value can be anything.

## Setting the correct product version

The aceDownloadUrl value in ace-minimal-image-pipeline-run.yaml is likely to need updating, either to another version
in the same server directory (if available) or else to an ACE developer edition URL from the IBM website. In the latter
case, start at https://www.ibm.com/docs/en/app-connect/13.0?topic=enterprise-download-ace-developer-edition-get-started
and proceed through the pages until the main download page with a link: 

![download page](ace-dev-edition-download.png)

The link is likely to be of the form
```
https://iwm.dhe.ibm.com/sdfdl/v2/regs2/mbford/Xa.2/Xb.WJL1cUPI9gANEhP8GuPD_qX1rj6x5R4yTUM7s_C2ue8/Xc.13.0.1.0-ACE-LINUX64-DEVELOPER.tar.gz/Xd./Xf.LpR.D1vk/Xg.12164875/Xi.swg-wmbfd/XY.regsrvs/XZ.pPVETUejcqPsVfDVKbdNu6IRpo4TkyKu/13.0.1.0-ACE-LINUX64-DEVELOPER.tar.gz
```
Copy that link into the aceDownloadUrl parameter, adjusting the version numbers in the other files as needed.

## Running the pipeline

Setting up the pipeline requires Tekton to be installed (which may already have happend via OpenShift operators, in which case
skip the first line), tasks to be created, and the pipeline itself to be configured. The following commands build the ace-minimal
image and push it to the registry:
```
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply -f tekton/minimal-image-build/01-ace-minimal-image-build-and-push-task.yaml
kubectl apply -f tekton/minimal-image-build/ace-minimal-image-pipeline.yaml
kubectl create -f tekton/minimal-image-build/ace-minimal-image-pipeline-run.yaml
tkn pipelinerun logs -L -f
```

The ace-minimal-build-image-pipeline builds not only the ace-minimal-build image but also
builds ace-minimal itself, and so the following can be run on their own to build both images:
```
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply -f tekton/minimal-image-build/01-ace-minimal-image-build-and-push-task.yaml
kubectl apply -f tekton/minimal-image-build/02-ace-minimal-build-image-build-and-push-task.yaml
kubectl apply -f tekton/minimal-image-build/ace-minimal-build-image-pipeline.yaml
kubectl create -f tekton/minimal-image-build/ace-minimal-build-image-pipeline-run.yaml
tkn pipelinerun logs -L -f
```

## OpenShift

Tekton is not normally installed directly on OpenShift, and the Red Hat OpenShift Pipelines operator
would be used instead. The majority of the other steps are the same, but the registry authentication is 
a little different: the namespace in which the pipeline and pod are running must match the project
name in the image registry tags. The examples show 
`image-registry.openshift-image-registry.svc.cluster.local:5000/ace/ace-minimal:13.0.1.0-alpine`,
which will work for the `ace` namespace.

To run outside the default namespace, a special SecurityContextConstraints definition must be created
and associated with the service account:
```
kubectl apply -f tekton/os/ace-scc.yaml
oc adm policy add-scc-to-user ace-scc -z ace-tekton-service-account
```
Without this change, errors of the form
```
task build-images has failed: pods "ace-minimal-image-pipeline-run-db8lw-build-images-pod" is forbidden: unable to validate against any security context constraint: 
```
may prevent the pipeline running correctly.

After that, the pipeline run files need to be adjusted to use the OpenShift registry, such 
as [ace-minimal-build-image-pipeline-run.yaml](ace-minimal-build-image-pipeline-run.yaml):
```
    - name: dockerRegistry
      # OpenShift
      value: "image-registry.openshift-image-registry.svc.cluster.local:5000/ace"
      #value: "quay.io/trevor_dolby"
      #value: "us.icr.io/ace-containers"
      #value: "aceDemoRegistry.azurecr.io"
      # Minikube
      #value: "192.168.49.2:5000/default"
```
and then the pipelines can be run as usual. The OpenShift Pipeline operator provides a 
web interface for the pipeline runs, which may be an easier way to view progress.