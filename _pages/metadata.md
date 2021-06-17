---
layout: page
title: Container Metadata Management
permalink: code/metadata/
subtitle: Container Registries as general blob storage is great but what manages all that metadata?
---

## TL;DR

## Some Background

I maintain an Open Source project called [Tern](https://github.com/vmware/tern) which generates a Software Bill of Materials (or SBOM for short) for container images. A good number of software components that get installed in a container image are individually downloaded rather than managed via a package manager. This makes it very difficult to figure out the intent of the person or organization that created the container image. It also doesn't give much information about who contributed to the container image. Most container images are based on previously existing container images. I figured this problem of managing container metadata could be solved if there was a "package manager" for container images. So I brought up the idea of "packaging" back at [KubeCon 2018](https://youtu.be/WY3s_cG9ia8), where I ask if a container image manifest could hold all this information such that a tool could make inferences about the container supply chain.

It turns out, the community had already been thinking about how to manage things like helm charts, OPA policies, filesystems, and signatures. This is how I had come to be involved in the [Open Container Initiative](https://opencontainers.org). At the time, it was understood that the container image does not need to be managed beyond identifying it by its digest. No dependency management is required as all the dependencies are taken into account. Updates are dealt with by rebuilding the container again and keeping a rolling tag which downstream consumers can pull from.

However, dependency management is not all that a package manager does. It provides valuable information about the supply chain to consumers and contributors. Some of these attributes are missing in the general container ecosystem. Two years and several supply chain attacks later, we are still having conversations about how best to do this. Here, I try to boil down some proposed concepts and see how they hold up to our requirements for metadata management. 

## Back to Basics

There are 3 reasons why one would write a package manager:

1. Identification - to give your new bundle of files a name and some other uniquely identifiable traits.
2. Context - to understand where your bundle sits in relation to the other bundles everyone else has put out there (i.e. dependency management).
3. Freshness - to make sure your bundle gets updated while maintaining its place in the ecosystem.

There is actually a 4th reason - Build Repeatability - which I think falls under "Context", but it is worth mentioning here because knowing the state of your bundle at a given time is necessary for you to be able to reason about a past "good" state and a present "not-so-good" state, i.e., bisecting a build.

The container image ecosystem has none of these features. A container can be "identified" by its digest. You don't need to manage an ecosystem as the whole ecosystem is already present in one unit. You don't need to update the container - just build a new one and whatever needs updating will hopefully get updated. This works as long as nothing changes under your application. However, if you plan to maintain your application long term, you must be prepared to deal with stack updates and major releases. We currently do not have a good way of managing stack updates beyond "pulling from latest". Breaking changes down the stack may block you from re-building the image, forcing you to keep a stale image around, because that image is known to work.

## Container Registries for Metadata Management

We could build a separate metadata storage solution, but container registries already exist. With some enhancements, they could be used to store supplemental metadata along with the container image. Organizations are already doing this with source images that contain source tarballs, and signature payloads such as what [cosign](https://github.com/sigstore/cosign) does. The nice thing about using registries is that the metadata can be stored along with the target container image. Proposals to the OCI specifications are mostly around structuring and referencing such data. This is basically package management on the server side. It is difficult to get these proposals in because the client-server relationship as it currently exists is very tight. Any change to the server impacts the client and any changes to the client's packaging mechanism affects the server. As a result, the OCI specifications as they stand cannot include enhancements beyond extending the current specification that is in sync with the way Docker clients and Docker registries currently work.

This article is not meant to criticize the state of the ecosystem right now, but rather to lead the conversation to a position where the community can maintain their container images more sustainably. A selfish reason on my part is to also demonstrate the need for package management in the container image space, even if container registries can support linking of related artifacts to container images, and links among container images themselves. Package Management in other ecosystems have a client-server relationship as well, so it's not new for the architecture to share the burden between client and server. The relationship, in this case, may just be a little different.

## Identification

A container image works somewhat like a [Merkle Tree](https://en.wikipedia.org/wiki/Merkle_tree) with 1 chunk of data, the [Image Config](https://github.com/opencontainers/image-spec/blob/master/config.md), and 1 or more chunks of data representing the container filesystem. Each of the chunks are hashed and placed in an [Image Manifest](https://github.com/opencontainers/image-spec/blob/master/manifest.md), which is also hashed.

<img src="{{site.baseurl}}/assets/img/containers/image_manifest.png">

Now, this is all well and good to specifically identify this bundle, but it is missing other identifiable traits, most notably, a name and version. In the container world, the name is the name of the repository and the version _could be_ a tag.

<img src="{{site.baseurl}}/assets/img/containers/image_manifest_with_tag.png">

Just like git, all references to file collections are hashes and other references can point to them like `HEAD` or `FETCH_HEAD` or a tag. The tag _could_ follow semantic versioning and _could_ be [moved to another commit](https://git-scm.com/docs/git-tag#Documentation/git-tag.txt--f). In the Open Source world, nobody does that because it's a sure way to piss off all of your collaborators. Container image tags do not seem to follow semantic versioning as a rule, but a lot of language package managers rely on it, so there may be some hope in tagging images with semantic versioning tags. Regardless, the hash is a pretty good identifier of the container image.

So what about other identifiable traits? Here are some traits that other package managers use:

- License and Copyright information
- Authors/Suppliers
- Date of release
- Pointers to source code
- List of third party components (starting OS, installed packages)

That last one can get pretty long as the number of actual third party components that contribute to creating a container image can run in the hundreds. Lately I have been advocating for putting all this information into an
SBOM (Software Bill of Materials). We've also been throwing around ways to sign the image. This could lead to a detached armored signature of the image manifest or a [signature payload](https://www.redhat.com/en/blog/container-image-signing). We now have multiple identification artifacts for our container image that we would like to link to it. The current OCI proposals uses "references". A "reference" is just a manifest containing the hash of
the blob and the hash of the thing it is referring to. In our case here, the reference is to the image manifest hash.

<img src="{{site.baseurl}}/assets/img/containers/image_references.png">

There are still some problems with this layout:

1. Registries only recognize artifacts that are tagged and hence will delete anything that is not tagged.
2. If the image is deleted, we now have dangling references.

The quick solution would be to tag all the manifests.

<img src="{{site.baseurl}}/assets/img/containers/image_references_with_tags.png">

This means something needs to be able to keep track of the tags and their relation to each other, along with keeping semantic versioning of all of the artifacts.

A longer term solution may be to define a canonical artifact manifest that the registry will recognize and treat as something special. If that is the case, then something needs to either count or keep track of the number of references associated with each manifest.

<img src="{{site.baseurl}}/assets/img/containers/artifact_manifest.png">

Registry implementers can keep track of the links in the diagrams in any way they want. For example, in this diagram, the number of references to each of the manifests are kept track of (minus the hash), and when the image manifest is deleted, the operation walks down the tree to the end of each of the references and deletes them in some order until the number of references is 0.

<img src="{{site.baseurl}}/assets/img/containers/reference_counting.png">

But here, we are getting into some sophisticated tracking system for all the related objects. This is synonymous with package management, except the management is either done by the registry or the client. An argument can be made to maintain a neutral garbage collector that can be used either by the registry or the client (the unix philosophy), but we're getting ahead of ourselves. Let's move on to the next problem.

## Context

You would think that a container image has no dependencies, and you would be right...for the container runtime. At build time though, the final container image does depend on the state of the starting container image, the FROM line in a Dockerfile if you will. Due to the magic of Merkle, the derived image has no connection to the previous image. As a result, all references to the old image need to be recreated for the new image, as well as adding additional artifacts.

<img src="{{site.baseurl}}/assets/img/containers/derived_image_new_tags.png">

The artifact manifest mechanism may have an advantage over the plain reference mechanism because the number of references are kept at a minimum while the artifact metadata gets updated.

<img src="{{site.baseurl}}/assets/img/containers/derived_image_updated_tags.png">

Both mechanisms support requirements such as supply chain security, chain of custody, and pedigree checks. Both mechanisms require reference management. In the former, a client would have to copy over the SBOM and signature manifest of the original image, update its references, and add new manifests. In the latter, a client would have to download the artifact manifests, augment them, and push them along with the new container image. Either way, a client needs to understand some syntax, either with the tag names or artifact construction and artifact types, not to mention some understanding of the ecosystem the artifacts came from (SBOMs and signatures are only two of the commonly used artifact types that could be included).

One use case I had been mulling over is how to link a collection of images together to describe a Kubernetes application for example. For example, the Jaegar application is actually a [collection of containers](https://artifacthub.io/packages/helm/jaegertracing/jaeger) that have their own dependency graph. We would like the registry to recognize these connections so the whole application can be transfered across registries along with their supplemental artifacts.

<img src="{{site.baseurl}}/assets/img/containers/multiple_images.png">

Here is where I am at the edge of my imagination (and my diagrams are getting too complex). But I hope these use cases illustrate that if a package manager is not needed now, it will be needed soon in order to manage these high level relationships.

## Updates

This is one place where I hope [Semantic Versioning](https://semver.org/) would be taken more seriously. Package managers use semantic versioning to allow for a range of versions that are compatible across the stack. This allows downstream consumers to accomodate updates with minimal disruption. As it stands, since container images can only be identified by their digest, the tags are the only way to indicate what version the container is at. This is where package management would be really helpful. Typically, a client queries a server to see if any packages required by an application has a new version. The client then tells the user that an update is available if they want to download it. If the user has a range of versions their application is compatible with, the client just downloads the new version within the range constraints.

The current distribution spec supports [listing tags](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#content-discovery). Beyond that, the registry offers nothing in the way of managing updates, and perhaps, this is all that is needed from the registry side. What that means is that the brunt of the work to check and pull updates falls on the client.

## Next Steps

The community is in agreement that these proposals are [quite large](https://groups.google.com/a/opencontainers.org/g/dev/c/6DGtH48fGOg) for the specification to accept at this time. However, there will (hopefully) be a set of public conversations that will try to break down these changes into ingestable bites such that it allows for backwards compatibility with the current state of affairs and moving the specification forward to address all of our concerns. Perhaps in the future, there would be no need for a package manager as the registry and the image or artifact format would take care of providing all of the data required to reason about the supply chain. But in the meantime, we need something that picks up the slack, i.e. a package manager.

## Acknowledgements

Many many thanks to @SteveLasker, @jdolitsky, @jonjohnsonjr, @dlorenc, @tianon, and @vbatts for their help in getting me to this level of understanding of the ecosystem over the course of 2 years.

PS: I don't have comments enabled, but you can reach me on [Twitter](https://twitter.com/nishakmr/) if you would like to have a discussion. I hope we can figure it out together :).
