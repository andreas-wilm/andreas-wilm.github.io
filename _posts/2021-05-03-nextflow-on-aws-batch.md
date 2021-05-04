---
layout: post
title: Nextflow on AWS Batch
#subtitle: Exploring AutoML
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
#tags: [test]
comments: true
---

After a [recent job change](https://www.linkedin.com/feed/update/urn:li:activity:6777638663535763456/), I'm back at science! Yay! 

With this new job come analysis demands for huge next-gen sequencing data-sets. For orchestration of the analysis workflow orchestration I started out using Snakemake, simply because it's easier to use than the alternatives, at least for me. However, I knew that I would have to move workflow execution to AWS Batch at some stage and furthermore I couldn't figure out how to create a fully self-contained container image for Snakemake (see also this [Stackoverflow post](https://stackoverflow.com/questions/67193421/snakemake-docker-image-with-all-executables-preinstalled)), so I switched to Nextflow, which is a very different (and very powerful) beast. Anyway, I experienced some hiccups while getting this to run on AWS Batch, which I will discuss here, together with some general considerations.

## Debugging with Cloudwatch

Debugging upstream problems that are actually system related and have nothing to do with Nextflow itself, can be very hard to debug. [Using Cloudwatch logs](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_cloudwatch_logs.html) is a must, otherwise you won't get useful messages for upstream errors that are caused by Docker, Batch itself, AMI specific problems etc. 


## The AWS CLI

One of the problems I encountered early was

    /usr/bin/python2: error while loading shared libraries: libpython2.7.so.1.0: cannot open shared object file: No such file or directory
    
What? Why? After all, AMI and container worked perfectly when testing it interactively.

There [is a closed Nextflow issue](https://github.com/nextflow-io/nextflow/issues/1116) that gives you a hint, that something is wrong with the AMI. Had I carefully read the Nextflow documentation, I would have known that the ["*AWS CLI tool must to be installed in your custom AMI by using a self-contained package manager such as Conda.*"](https://www.nextflow.io/docs/latest/awscloud.html#aws-cli-installation). Instead, I had installed the AWS CLI via the system package manager.

Once the AWS CLI was installed via Miniconda and `batch.cliPath` was set as described in the docs, the problem disappeared. Having said this, it's a really non-intuitive error message that can send you down deep rabbit holes.

## Disks and Docker

Since we have to analyse hundreds of GB per sample, sufficient disk storage is a must. The easiest would be to make the root disk large enough (say 1TB EBS) during AMI creation. Now, do you need to change some Docker settings to make use of this? Some some older blog posts and confusingly even some recent AWS posts mention that you should change `dm.basesize` to avoid "No space left on device" errors (see e.g. [here](https://aws.amazon.com/premiumsupport/knowledge-center/batch-job-failure-disk-space/)). However this is not needed for Amazon Linux v2, which uses overlayfs and makes things a lot easier.

Unfortunately, even with a large enough disk you might run into problems, that seem at first sight not related to storage at all, most importantly `CannotInspectContainerError`, which is basically a Docker timeout. Why is this happening here? With GB of data being staged in (and out) from S3 and Docker images being pulled, the disk IO can get saturated quite easily, especially if you're using EBS. There are EBS disk types with provisioned IOPS, but it will be hard to predict your needs. So why not use ephemeral disks / local instance store instead? We used this solution already for the National Precision Medicine pilot SG10K three years ago. The disadvantage is that you obviously need to limit yourself to instance types with local disks and furthermore that the disks need to be initialized at startup. In my case I opted for instance types with two NVMe disks (m5d.8xlarge) and combined them as a 1TB RAID0 during startup (see also the [AWS docs on how to configure RAID](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/raid-config.html)). This [older Gist](https://gist.github.com/andreas-wilm/3460a788d6548370a136e63b5b91281e) shows how to determine available disks. After the RAID0 initialization, the bootup script creates the FS and mounts the disk as `/var/lib/docker`. The alternative to mounting is apparently to set `data-root` in `/etc/docker/daemon.json`, which I couldn't get to work though.


## OutOfMemoryError: Container killed due to memory usage

Once running on AWS Batch, even simple jobs (like MD5sum checks on FastQ files) started failing with `OutOfMemoryError` and needed their memory requirement bumped up to 8GB, which was weird. Two seemingly unrelated Nextflow issues ([S3 Download Fails when using AWS Batch #1107 ](https://github.com/nextflow-io/nextflow/issues/1107#) and [Job submission rate throttling needed for AWS Batch #1371](https://github.com/nextflow-io/nextflow/issues/1371)) made me try the following settings:

    client.maxConnections = 4
    batch.maxParallelTransfers = 8
    
and now the jobs work (at least with 4GB only). I don't fully understand this (cargo cult alarm), but won't touch it again until it breaks :)


## Notes on AWS permissions

By now there are a few "Nextflow on AWS Batch" posts out there. They all give you a very good idea about the steps required. However, they are also very, let's say, "generous" with the AWS permissions given to the user account. I've seen `AmazonEC2FullAccess`, `AmazonS3FullAccess` and `AWSBatchFullAccess` mentioned as the default. **Don't do this!** Not even on an academic, AWS sponsored account :) Follow the principle of least privilege, even if annoying at first, i.e. limit permissions as much as you can.

For S3 you can create custom policies that only allow reading from / writing to certain buckets. 

For ECS you really only need the AWS managed policy `AmazonEC2ContainerServiceforEC2Role`.

And for AWS Batch you only need to allow the following actions (instead of full access), which again should be limited to corresponding resources:

- `batch:TerminateJob`,
- `batch:DescribeJobs`,
- `batch:DescribeJobDefinitions`,
- `batch:DescribeJobQueues`,
- `batch:RegisterJobDefinition`,
- `batch:SubmitJob` 


## Alternatives

[Luke Goodsell](https://twitter.com/luke_goodsell) mentioned an alternative setup to local instance store with RAID0: they "*use the @SeqeraLabs
 Enterprise extension for Nextflow for the shared fs (and excellent support)*". See [this Twitter thread](https://twitter.com/me_myself_andY/status/1388021305509781511) for more information.

