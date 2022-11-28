### Introduction

This demo shows how to use [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable) with OpenShift Pipelines (i.e. Tekton) to manage the promotion of a new image across environments.

The way this demo works is that Argo CD Image Updater is configured to monitor an image repository for changes in environment tags, i.e. `dev`, `test` and `prod`, and when this happens to update the manifests in git to reference the new image via its digest. This writeback is done using the `kustomize edit set image` to have [kustomize](https://kustomize.io) override the value in the deployment manifest.

From a pipelines point of view once the image is built the following steps are taken:
1. Tag the new image with the appropriate environment tag
2. Wait for the Argo CD Image Updater to write the image reference and for Argo to deploy it
3. Run an automated test (using newman) to validate the application deployment

Note that step #2 is performed with a custom task that monitors the image currently deployed and waits for it to match the digest of the new image.

### Pre-requisites

1. You will need kustomize and helm deployed on your local machine
2. An account in GitHub in order to fork this repository
3. An account in a registry, quay.io for example, where you can create an image repository

### Steps to deploy

1. Fork this repository

2. Install the OpenShift GitOps operator via Operator Hub

3. Install the community Argo CD Image Updater:

```
kubectl apply -n openshift-gitops -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

4. Update the permissions for the image updater in the argocd CR in openshift-gitops:

```
p, role:image-updater, applications, get, */*, allow
p, role:image-updater, applications, update, */*, allow
g, image-updater, role:image-updater
```

5. Install the OpenShift Pipelines operator

6. Create a repository in an accessible registry, quay.io allows you to create free public repositories

7. In your fork, modify the repository in `bootstrap/argocd/base/values.yaml` to reflect your forked repository and your forked registry, in particular change:

* Change the repoURL from `https://github.com/gitops-examples/argocd-image-updater-example` to your forked repo
* Change the value of `argocd-image-updater.argoproj.io/image-list` from `quay.io/gnunn/server:<tag>` to your registry, i.e. `quay.io/<your-user-name>/server`
* Do NOT change annotation `argocd-image-updater.argoproj.io/server.kustomize.image-name` since this is the image referenced in the deployment.

8. Deploy the applications via OpenShift GitOps by running the bootstrap script:

```.bootstrap.sh```

9. Create a docker secret for the registry in step #8 in the `demo-cicd` namespace so that the pipeline can access your repository, first create a .dockerconfigjson file, here is an example:

```
{
	"auths": {
		"quay.io": {
		  "auth": "XXXXXXXXXX"
		}
	}
}
```

Now create a secret out of that file:

```
oc create secret docker-registry container-registry --from-file=.dockerconfigjson -n demo-cicd
```

10. Link this secret to the pipelines service account:

```
oc secrets link pipeline container-registry --for=pull,mount -n demo-cicd
```

11. Add your gitops credentials to the demo-cicd namespace so that the argocd-image-updater can update the git repository:

```
oc -n argocd-image-updater create secret generic git-creds \
  --from-literal=username=<username> \
  --from-literal=password=<password or PAT> -n demo-cicd
```