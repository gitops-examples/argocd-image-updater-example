(Demo still a work in progress, do not use)

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

6. In your fork, modify the repository in `bootstrap/argocd/base/values.yaml` to reflect your forked repository

7. Deploy the applications via OpenShift GitOps by running the bootstrap script:

```.bootstrap.sh```

8. Create a repository in an accessible registry, quay.io allows you to create free public repositories

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