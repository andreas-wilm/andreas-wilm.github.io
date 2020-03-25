---
layout: post
title: Singularity and Azure Container Registry
#subtitle: Exploring AutoML
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
#tags: [test]
comments: true
---

Since April 2019 [Azure Container Registry (ACR) supports storing of Singularity images](https://azure.microsoft.com/en-us/blog/azure-container-registry-now-supports-singularity-image-format-containers/). Today was the first time I actually had to use it and here I quickly discuss how.

Once I had [installed Singularity](https://sylabs.io/guides/3.5/admin-guide/installation.html#), I pulled an example image from Singularity's cloud library. To push this image to ACR, you obviously need to authenticate. For Docker you would normally use `docker login` or `az acr login` followed by `docker push`. This works differently for Singularity. One option is to use ORAS  for log-in **and** uploading. See the [Microsoft Docs](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-oci-artifacts) for how to do this.

However, I wanted use the `singularity` command directly for pushing. For this you need to use a docker username and password for this. But where do you get this from in case of ACR? Turns out you need a [Service Principal](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication#service-principal). Follow [these instructions](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal) on how to create a new one or reuse an existing one. Make sure to use the `acrpush` permission (allowing push and pull) and modify `ACR_NAME` and `SERVICE_PRINCIPAL_NAME` as needed. Note that `ACR_NAME` is just the repository name and not the full login server URL.

Now you can push (or pull) using the `--docker-username` and `--docker-password` options:

    singularity push --docker-username <spuser> --docker-password <sppass> image.sif oras://registry/namespace/image:tag

where `spuser` and `sppass` are the returned Service Principal username and password and `registry` is the repository login server.

And that's it.






