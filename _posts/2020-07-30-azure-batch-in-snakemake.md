---
layout: post
title: Integrating Azure Batch support in Snakemake
#subtitle: Exploring AutoML
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
#tags: [test]
comments: true
---

In my two previous posts I showed how to run Snakemake on Azure Kubernetes Service (AKS) without a shared filesystem. While there is nothing wrong with this setup, there is another Azure service that's actually meant for this type of batch computing scenario: [Azure Batch](https://docs.microsoft.com/en-us/azure/batch/), a cloud based job scheduling service for efficiently running large-scale parallel and high performance computing applications. [BizData](https://www.bizdata.com.au/) has an advanced commercial offering (including support) for Snakemake, which builds on [Batch Shipyard](https://batch-shipyard.readthedocs.io/) and is called [Genomics Pipeline Acceleration on Demand](https://www.bizdata.com.au/genomics-ondemand) (see [SnakemakeBurst](https://github.com/Azure/azure-hpc/tree/master/LifeSciences/SnakemakeBurst) for more info). Their solution requires only minor modifications of your Snakefile and offers all sorts of bells and whistles. 

Native Azure Batch integration in Snakemake is however missing. Recently native Google Cloud Life Sciences support was  built into Snakemake (amazing work by [Vanessa Sochat](https://vsoch.github.io/)) and with that I had a template to follow for native Azure Batch implementation (think of it as a poor man's version of the BizData solution). And during the [Microsoft Hackathon 2020](https://twitter.com/MSFTGarage/status/1288666876432642050) I finally had the chance to work on this.

The code can be found on the [azbatch branch](https://github.com/andreas-wilm/snakemake/tree/azbatch) of my Snakemake fork, which builds on my previous (at the time of writing still not merged) ["Azure as default remote provider" pull request](https://github.com/snakemake/snakemake/pull/324).

As expected, three days were not enough to make this a polished implementation, i.e. this is still work in progress, but it actually does work. In this post I will to document the status, hoping that someone else can polish this work, in case I don't find the time to complete it. Do contact me if you are interested to complete this.


# Overview


The main work went into `executors/azure_batch.py`, which is largely modeled after the Google Cloud Life Science equivalent `executors/google_lifesciences.py`. This defines the AzBatchExecutor class and AzBatchJob (actually a task in Azure Batch parlance).

Given a Batch and Storage account, the overall procedure is as follows:

- pack workflow sources (config files, Snakefile etc.) and upload to blob
- create an Azure Batch compute pool, which automatically pulls the required Docker image 
- create an Azure Batch job (actually a group of tasks)
- create an Azure Batch task for each Snakemake job to run. For each task:
  - workflow sources are downloaded
  - data is automatically staged in and out to blob (no shared filesystem!)
- monitor tasks for completion
- upon shutdown, delete the Azure Batch pool and job

My current implementation (commit 98b49c8a) does all of the above. However, there are some remaining issues:

- Azure Batch parameters are provided to Snakemake using a yaml file with `--az-batch-config`. Instead separate arguments should be used.
- Batch autoscaling doesn't work. I gave this only a quick try using one of the standard formulas, but the pool never increased from 0. Therefore autoscaling is currently disabled (see `create_pool()`)
- The workflow resources are not deleted from blob during `shutdown()`
- By default Azure Batch only places one job per node, which is inefficient
- The Azure Batch retry option is hardwired to two (see `create_job()`)
- Lots of minor issues marked with `FIXME` in the [code](https://github.com/andreas-wilm/snakemake/blob/azbatch/snakemake/executors/azure_batch.py)
- Untested: when using instances with extra disks, is the space made available in Docker?
- Untested: use of spot instances (that's right and yes they are different from low prio VMs)


# Installation and Running

For the brave, here are installation instructions. First install [my azbatch branch](https://github.com/andreas-wilm/snakemake/tree/azbatch). Refer to notes in my [previous post]({{ site.baseurl }}{% link _posts/2020-06-08-snakemake-on-ask.md %}) on how to do that with conda (but be sure to use the azbatch branch here).
 
Then install the required modules `azure-storage-blob` and `azure-batch` in your Snakemake environment.

Next create a blob storage and Batch account on Azure.

As mentioned, the current implementation receives all Azure Batch parameters in a yaml file (not good), which looks as follows:

    BATCH_ACCOUNT_NAME: 'Your-batch-account-name'
    BATCH_ACCOUNT_KEY: 'Your-Batch-account-key'
    BATCH_ACCOUNT_URL: 'Your-Batch-account-URL'
    BATCH_POOL_NODE_COUNT: 3# Pool node count
    BATCH_POOL_VM_SIZE: 'Standard_D3_v2'# VM Type/Size

Create that file (called `yourazbatch.yaml` below) and modify as needed. 

To make Snakemake aware of the Blob account use the following and use `--envvars` accordingly (see below):

    export AZ_BLOB_ACCOUNT_URL="blob url with SAS"

The URL **must** contain a SAS. The additional `AZ_BLOB_CREDENTIAL` (see previous post) is not needed and will not work!

Then run Snakemake with e.g.:

    snakemake --use-conda \
        --default-remote-prefix yourcontainer \
        --default-remote-provider AzBlob \
        --az-batch-config yourazbatch.yaml \
        --envvars AZ_BLOB_ACCOUNT_URL \
        --container-image andreaswilm/snakemake:az-batch-20200630
        --jobs 3 --reason --forceall --verbose   

The referenced container image contains my current version of the code and is hosted on Dockerhub. If you want to modify the code, add `pip install azure-storage-blob azure-batch` to the Dockerfile before you build a new image.

Note, for production code it's best if the image contains all your software preinstalled, rather than installing it with conda on the fly.


