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

6. In your fork, modify the repository in `bootstrap/argocd/base/values.yaml` to reflect your repository

7. Deploy the applications via OpenShift GitOps by running the bootstrap script:

```.bootstrap.sh```