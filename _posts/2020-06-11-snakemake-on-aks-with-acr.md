---
layout: post
title: Using Azure Container Registries with Snakemake and AKS
#subtitle: Exploring AutoML
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
#tags: [test]
comments: true
---

In a [previous post]({{ site.baseurl }}{% link _posts/2020-06-08-snakemake-on-ask.md %}) I showed how to run Snakemake on an auto-scaling Kubernetes cluster without shared filesystem on Azure. There I used a public Dockerhub repo for the Snakemake container. Since we were running Kubernetes on Azure it makes sense to use Azure Container registries (ACR; which supports [Singularity images](https://azure.microsoft.com/en-us/blog/azure-container-registry-now-supports-singularity-image-format-containers/) by the way). But how do you authenticate with ACR from within your Kubernetes cluster? Easy, you can simply attach an ACR repo to a Kubernetes cluster [during creation](https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?toc=/azure/container-registry/toc.json&bc=/azure/container-registry/breadcrumb/toc.json#create-a-new-aks-cluster-with-acr-integration) or [afterwards](https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?toc=/azure/container-registry/toc.json&bc=/azure/container-registry/breadcrumb/toc.json#configure-acr-integration-for-existing-aks-clusters) if you have an existing cluster. Below I will go with the former option and create a new cluster.

First, create an ACR repository:

    MYACR=snakemaksacr;# change to something unique
    # assuming resource group snakemaks-rg still exists
    az acr create -n $MYACR -g snakemaks-rg --sku basic

Next, import the Docker image from Dockerhub to ACR:

    az acr import -n $MYACR --source docker.io/andreaswilm/snakemaks:5.17 --image snakemaks:5.17

Then, create a new AKS cluster with the above ACR repo attached:

    az aks create --resource-group snakemaks-rg --name snakemaks-aks \
     --vm-set-type VirtualMachineScaleSets --load-balancer-sku standard --enable-cluster-autoscaler \
     --node-count 1 --min-count 1 --max-count 3 --node-vm-size Standard_D3_v2 \
     --generate-ssh-keys --attach-acr $MYACR

The only new options required are `--generate-ssh-keys --attach-acr $MYACR`.

Assuming you kept the setup from the previous post (conda environment and storage account with snakemake tutorial data) you can run Snakemake after downloading the Kubernetes credentials:

    az aks get-credentials --resource-group snakemaks-rg --name snakemaks-aks
    snakemake --kubernetes --container-image ${MYACR}.azurecr.io/snakemaks:5.17 \
      --default-remote-prefix snakemake-tutorial --default-remote-provider AzBlob \
      --envvars AZ_BLOB_ACCOUNT_URL AZ_BLOB_CREDENTIAL \
      --use-conda --jobs 3
      
Only the container URI has changed. Note, that if the results of the previous run still exist, you might want to add `--forceall` to the Snakemake call to make sure it does something.

