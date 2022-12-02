### Introduction

This demo shows how to use community [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable) with OpenShift Pipelines (i.e. Tekton) to manage the promotion of a new image across environments.

The way this demo works is that Argo CD Image Updater is configured to monitor an image repository for changes in environment tags, i.e. `dev`, `test` and `prod`, and when this happens to update the manifests in git to reference the new image via its digest. This writeback is done using the `kustomize edit set image` to have [kustomize](https://kustomize.io) override the value in the deployment manifest.

From a pipelines point of view once the image is built the following steps are taken:
1. Tag the new image with the appropriate environment tag
2. Wait for the Argo CD Image Updater to write the image reference and for Argo to deploy it
3. Run an automated test (using newman) to validate the application deployment

Note that step #2 is performed with a custom [task](components/tekton/tasks/base/argocd-image-updater-wait.yaml) that monitors the image currently deployed and waits for it to match the digest of the new image.

### Prerequisites

1. You will need kustomize and helm deployed on your local machine
2. An account in GitHub in order to fork this repository
3. An account in a registry, quay.io for example, where you can create an image repository

### Steps to deploy

Here are the steps required to install this demo, note this is assuming an empty cluster (such as one from RHPDS).

1. Fork this repository

2. Install the OpenShift GitOps operator via Operator Hub

3. Install the community Argo CD Image Updater:

```
kubectl apply -n openshift-gitops -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

4. Update the permissions for the image updater in the argocd CR in openshift-gitops:

```
	rbac:
	  policy: |
		...
		p, role:image-updater, applications, get, */*, allow
		p, role:image-updater, applications, update, */*, allow
		g, image-updater, role:image-updater
```

Ignore the warning about this being a managed resource.

5. Install the OpenShift Pipelines operator

6. Create a repository in an accessible registry, quay.io allows you to create free public repositories

7. In your fork, modify the repository in `bootstrap/argocd/base/values.yaml` to reflect your forked repository and your forked registry, in particular change:

	* Change the repoURL from `https://github.com/gitops-examples/argocd-image-updater-example` to your forked repo
	* Change the value of `argocd-image-updater.argoproj.io/image-list` from `quay.io/gnunn/server:<tag>` to your registry, i.e. `quay.io/<your-user-name>/server`
	* Do NOT change annotation `argocd-image-updater.argoproj.io/server.kustomize.image-name` since this is the image referenced in the deployment.

8. Deploy the applications via OpenShift GitOps by running the bootstrap script:

```./bootstrap.sh```

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

Note that the `XXXXXXX` part is a base64 of "<username>:<password>" using your username/password for the registry. You can create this in Linux with the following command:

```
echo -n "<username>:<password>" | base64"
```
And then taking the string that is output and replacing auth with it. Now create a secret out of that file:

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

### Validate the Deployment

1. Navigate to your Argo CD and verify that all of the applications are deployed and are in sync.

![alt text](https://raw.githubusercontent.com/gitops-examples/argocd-image-updater-example/main/docs/img/argo-cd-app-status.png)

Note that the demo-cicd project may show progressing because the PVCs are in a pending state waiting for them to be bound when you execute the pipeline. You can correct this by modifing the PVC health check in Argo CD but you will only notice this prior to the first run.

![alt text](https://raw.githubusercontent.com/gitops-examples/argocd-image-updater-example/main/docs/img/demo-cicd-progressing-details.png)

### Execute the Pipeline

1. Navigate to the `demo-cicd` project and select Pipelines in the console.

2. You should see one pipeline, click the kebab menu and select `Start`

3. A new window will appear for Pipeline parameters, change the following items:

	* Change `image_dest_url` to point to your image repository
	* Set all of the workspaces as per the image below.

![alt text](https://raw.githubusercontent.com/gitops-examples/argocd-image-updater-example/main/docs/img/pipeline-workspaces.png)

4. Press `Start` at the bottom of the pipeline to execute it.

	* Note the first time you run the pipeline the build step will take time to execute since it is downloading all of the dependencies from maven repositories. These are cached in a PVC so subsequent executions should be much faster

5. If everything went to plan you should see the pipeline successfully execute:

![alt text](https://raw.githubusercontent.com/gitops-examples/argocd-image-updater-example/main/docs/img/pipeline-success.png)

### Optimizations

1. Note that it can take some time for Image Updater to pick up the tag change in the registry and then more time for Argo CD to deploy the changed manifests. You can optimize this a bit by setting up a webhook between the manifest repository `argocd-image-updater-example` and Argo CD. This removes the time waiting for Argo CD to poll the repository. There doesn't appear to be a way to have Argo CD Image Updater accept a webhook from the registry at this time per this [issue](https://github.com/argoproj-labs/argocd-image-updater/issues/1).

2. The first time I run this with quay.io I'm briefly getting a degraded deployment as the digest cannot be pulled, need to investigate this further. Subsequent runs don't have this issue.
