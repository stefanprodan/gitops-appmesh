# gitops-appmesh

Progressive Delivery on EKS with AppMesh, Flagger and Flux v2.

## Prerequisites

You will need a EKS cluster version 1.16 or newer and kubectl version 1.18.

Install [eksctl](https://eksctl.io/) and the [Flux](https://fluxcd.io) CLI:

```sh
brew install eksctl fluxcd/tap/flux
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

## Cluster bootstrap

Create an EKS cluster with AppMesh IAM role in the us-west-2 region:

```sh
cat << EOF | eksctl create cluster -f -
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: appmesh
  region: us-west-2
nodeGroups:
  - name: default
    instanceType: m5.large
    desiredCapacity: 2
    volumeSize: 120
    iam:
      withAddonPolicies:
        appMesh: true
        xRay: true
        certManager: true
        albIngress: true
EOF
```

Verify that your EKS cluster satisfies the prerequisites with:

```console
$ flux check --pre
► checking prerequisites
✔ kubectl 1.19.4 >=1.18.0
✔ Kubernetes 1.17.12-eks-7684af >=1.16.0
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
