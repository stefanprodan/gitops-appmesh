# gitops-appmesh

Progressive Delivery on EKS with AppMesh, Flagger and Flux v2.

## Prerequisites

Install [eksctl](https://eksctl.io/), [yq](https://mikefarah.gitbook.io/yq/)
and the [Flux](https://github.com/fluxcd/flux2) CLI:

```sh
brew install eksctl yq fluxcd/tap/flux
```

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Fork this repository on your personal GitHub account and export your access token, username and repo:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=gitops-appmesh
```

Clone the repository on your local machine:

```sh
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

## Cluster bootstrap

Create a cluster with eksctl:

```sh
eksctl create cluster -f .eksctl/config.yaml
```

The above command with create a Kubernetes cluster v1.18 with two `m5.large` nodes in the us-west-2 region.

Verify that your EKS cluster satisfies the prerequisites with:

```console
$ flux check --pre
► checking prerequisites
✔ kubectl 1.19.4 >=1.18.0
✔ Kubernetes 1.18.9-eks-d1db3c >=1.16.0
✔ prerequisites checks passed
```

Install Flux on your cluster with:

```sh
flux bootstrap github \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/appmesh
```

The bootstrap command commits the manifests for the Flux components in `clusters/appmesh/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Wait for the cluster reconciliation to finish:

```console
$ watch flux get kustomizations 
NAME              	REVISION                                     	READY
appmesh-controller	main/6dddb26ac0b07a0557dd930da9f29186f0c155c4	True
appmesh-gateway   	main/6dddb26ac0b07a0557dd930da9f29186f0c155c4	True
appmesh-prometheus	main/6dddb26ac0b07a0557dd930da9f29186f0c155c4	True
apps              	main/6dddb26ac0b07a0557dd930da9f29186f0c155c4	True
flagger           	main/6dddb26ac0b07a0557dd930da9f29186f0c155c4	True
flux-system       	main/6dddb26ac0b07a0557dd930da9f29186f0c155c4	True
sources           	main/6dddb26ac0b07a0557dd930da9f29186f0c155c4	True
```

Verify that Flagger, Prometheus, AppMesh controller and gateway Helm releases have been installed:

```console
$ flux get helmreleases --all-namespaces 
NAMESPACE      	NAME              	REVISION	READY
appmesh-gateway	appmesh-gateway   	0.1.5   	True
appmesh-system 	appmesh-controller	1.2.0   	True
appmesh-system 	appmesh-prometheus	1.0.0   	True
appmesh-system 	flagger           	1.2.0   	True
```

## Application bootstrap

Wait for Flagger to initialize the canary:

```console
$ watch kubectl -n apps get canary
NAME      STATUS      WEIGHT   LASTTRANSITIONTIME
podinfo   Initialized 0        2020-11-14T12:03:39Z
```

Find the AppMesh Gateway public address with:

```sh
export URL="http://$(kubectl -n appmesh-gateway get svc/appmesh-gateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
echo $URL
```

Wait for the DNS to propagate and  podinfo to become accessible:

```console
$ watch curl -s ${URL}
{
  "hostname": "podinfo-primary-5cf44b9799-lgq79",
  "version": "5.0.0"
}
```

When the URL becomes available, open it in a browser and you'll see the podinfo UI.

## Automated canary promotion

When you deploy a new podinfo version, Flagger gradually shifts traffic to the canary,
and at the same time, measures the requests success rate as well as the average response duration.
Based on an analysis of these App Mesh provided metrics, a canary deployment is either promoted or rolled back.

First pull the changes from GitHub:

```sh
git pull origin main
```

Bump podinfo version from `5.0.0` to `5.0.1`:

```sh
yq w -i ./apps/kustomization.yaml images[0].newTag 5.0.1
```

Commit and push changes:

```sh
git add -A && \
git commit -m "podinfo 5.0.1" && \
git push origin main
```

Tell Flux to pull the changes or wait one minute for Flux to detect the changes:

```sh
flux reconcile source git flux-system
```

Wait for the cluster reconciliation to finish:

```sh
watch flux get kustomizations
```

When Flagger detects that the deployment revision changed, it will start a new rollout.
You can monitor the traffic shifting with:

```sh
watch kubectl -n apps get canary
```

Watch Flagger logs:

```console
$ kubectl -n appmesh-system logs deployment/flagger -f | jq .msg
New revision detected! Scaling up podinfo.apps
Starting canary analysis for podinfo.apps
Pre-rollout check acceptance-test passed
Advance podinfo.apps canary weight 5
Advance podinfo.apps canary weight 10
Advance podinfo.apps canary weight 15
Advance podinfo.apps canary weight 20
Advance podinfo.apps canary weight 25
Advance podinfo.apps canary weight 30
Advance podinfo.apps canary weight 35
Advance podinfo.apps canary weight 40
Advance podinfo.apps canary weight 45
Advance podinfo.apps canary weight 50
Copying podinfo.apps template spec to podinfo-primary.apps
Routing all traffic to primary
Promotion completed! Scaling down podinfo.apps
```

Lastly, open up podinfo in the browser. You'll see that as Flagger shifts more traffic
to the canary according to the policy in the Canary object,
we see requests going to our new version of the app.

## Cleanup

Suspend the cluster reconciliation:

```sh
flux suspend kustomization cluster-addons
```

Delete the AppMesh objects:

```sh
kubectl delete mesh --all
```

Delete the EKS cluster:

```sh
eksctl delete cluster -f .eksctl/config.yaml
```
