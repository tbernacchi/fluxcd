# FluxCD 

This is a simple fluxcd repository with a simple app to understand the tool.

## Requirements 
* Kubernetes cluster;
* Github repo;
* Buildah;

## Kubernetes cluster 
For the k8s cluster I used [Vagrant](https://www.vagrantup.com/) for the environment, more specific this [one](https://github.com/tbernacchi/vagrant/blob/master/kubeadm-2023/README.md) running on Ubuntu 20.04.6 LTS. If for some reason the `vagrant up` does not finish as expected I've put all the scripts necessaries for the bootstrap of the cluster on this [repo](https://github.com/tbernacchi/scripts-bash/tree/main/scripts/kubeadm-2023). Run in this order and you'll be good to go: [sysctl.sh](https://github.com/tbernacchi/scripts-bash/blob/main/scripts/kubeadm-2023/sysctl.sh) [containerd.sh](https://github.com/tbernacchi/scripts-bash/tree/main/scripts/kubeadm-2023) [runc.sh](https://github.com/tbernacchi/scripts-bash/blob/main/scripts/kubeadm-2023/runc.sh) [k8s.sh](https://github.com/tbernacchi/scripts-bash/blob/main/scripts/kubeadm-2023/k8s.sh).


## How to install

On the Kubernetes cluster:

```
curl -s https://fluxcd.io/install.sh | sudo FLUX_VERSION=2.1.2 bash
mkdir -p fluxcd/clusters/dev-cluster
```

You can also check more ways to install [here](https://fluxcd.io/flux/installation/).

To bootstrap the fluxcd:

```
flux bootstrap github --token-auth --owner=tbernacchi --repository=fluxcd --branch=main --path=clusters/dev-cluster --personal --private=false
```


If everything went well you'll see the new CRD on your cluster:

```
# kubectl api-resources | grep -i git
gitrepositories                   gitrepo      source.toolkit.fluxcd.io/v1              true         GitRepository
```

At this point `fluxcd` will create some files on the repository you've set, you can check running `git pull` of your repo:

```git pull
remote: Enumerating objects: 11, done.
remote: Counting objects: 100% (11/11), done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 10 (delta 0), reused 10 (delta 0), pack-reused 0
Unpacking objects: 100% (10/10), 35.62 KiB | 5.94 MiB/s, done.
From github.com:tbernacchi/fluxcd
   13f5dc2..45e33c6  main       -> origin/main
Updating 13f5dc2..45e33c6
Fast-forward
 infra-repo/k8s/clusters/dev-cluster/flux-system/gotk-components.yaml | 8029 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 1 file changed, 8029 insertions(+)
 create mode 100644 infra-repo/k8s/clusters/dev-cluster/flux-system/gotk-components.yaml
```
To check the status of flux:

```
flux check
```

## Building the app image

For this step I've use [Buildah](https://buildah.io/) for it. If you don't have Buildah you can use [Docker](https://docs.docker.com/engine/reference/commandline/build/) as well.  
To install Buildah on Ubuntu 20.04.6 I've use this [repo](https://gist.github.com/sebastianwebber/2c1e9c7df97e05479f22a0d13c00aeca) as a reference and put the steps into this [script](https://github.com/tbernacchi/scripts-bash/blob/main/scripts/kubeadm-2023/install-buildah.sh).


Building the image from the repo directory:

```
~/fluxcd# buildah bud -t example-app-1:0.0.1 app/src/dockerfile
```

## Github container registry

For this lab I've used [Github](https://docs.github.com/en/packages/working-with-a-github-packages-registry/) container registry to store the containers images. </br>
You also are going to need to create a token as it's mentioned [here](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens).

```
export GITHUB_TOKEN=<your-beautiful-token>
```

Buildah login with your GITHUB_TOKEN:

```
buildah login ghcr.io
Username: tbernacchi
Password:
Login Succeeded!
```

Let's tag and push the image:

Tag:

```
buildah tag localhost/example-app-1:0.0.1  ghcr.io/tbernacchi/fluxcd/example-app-1:0.0.1
```

Push:

```
buildah push ghcr.io/tbernacchi/fluxcd/example-app-1:0.0.1
```

## Fluxcd 

On the the k8s cluster let's apply the CRD's to tell flux where our Git repo is and where the YAML is.

```
~/fluxcd# tree infra-repo/apps/
infra-repo/apps/
└── example-app-1
    ├── gitrepository.yaml
    └── kustomization.yaml

1 directory, 2 files
```

```
kubectl apply -f infra-repo/apps/example-app-1/gitrepository.yaml
kubectl apply -f infra-repo/apps/example-app-1/kustomization.yaml

# Check flux resources
kubectl get GitRepository -n flux-system 
kubectl get Kustomization -n flux-system

# check deployed resources
kubectl get all
```

```
#To access the app
kubectl port-forward svc/example-app-1 80:80
```

That's it. When you change your `app/example-app-1/deploy/deployment.yaml` file fluxcd will deploy the app. </br>
This was a basic CD setup.

## More information
https://fluxcd.io/ </br> 
https://fluxcd.io/flux/cmd/flux_bootstrap/ </br> 
https://fluxcd.io/flux/installation/bootstrap/github/
