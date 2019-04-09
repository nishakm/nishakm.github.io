---
title: Are My Good Intentions Paving a Road to Hell?
desc: Containerizing a Python script that was never meant to be containerized
---

# TL;DR

* Due to the overwhelming feedback from many product teams that [Tern](http://github.com/vmware/tern) needed to run in a container or as a web service or in a [Kubernetes](https://kubernetes.io/) cluster, I worked on making it run successfully in a [Docker](https://www.docker.com/) container.
* Tern primarily uses `mount`, which cannot be run from within the container without giving the container elevated privileges.
* Tern also uses `overlay2` mount which cannot be used within a running container (as the filesystem type is `overlay`). So an external bind mount is needed for mounting the `overlay2` filesystem.
* This is *really bad* for security. That's it. I mean, I write about escape plans at length but really, a system level script should not be running in a container.

# On being the maintainer of a boring project

Developing and maintaining an open source project to inspect containers for licenses (along with other metadata) is what I do at my day job. It requires some knowledge of container internals. I didn't expect it to in the beginning but I've been at this for almost 2 years now, and during that time, I've learned a lot about the Linux Kernel, SELinux, and various other system administration things. So needless to say, I've had a surprisingly good time working on a license compliance tool.

One of my job duties is to get individual developers or teams to use Tern. It's been an uphill battle to say the least. Not only do folks confuse Tern with a file scanner (it isn't a file scanner for...reasons), it's generally an uphill battle to get anyone to care about license compliance. So I spend a good bit of my time reaching out to teams. I'll go into how harrowing this is for me in another blog post. But for now, let's talk about the feedback I got.

A good number of teams asked if Tern could run in a container. To be honest, this took me by surprise. It shouldn't have as nowadays, everything is being run on the cloud. I mean *everything*. Even the things that you should run on your machine, that you use for development, or even building the things that run on someone else's machine, have to run in a container somewhere in the cloud, controlled by some container orchestrator. In some ways, this makes sense as creating web end points for random folks to access is very attractive. In other ways, I don't see the point of running a pre-deployment tool in a post-deploy environment unless it was a pre-deploy sandbox cloud (on-prem). But what do I know? So whatever, I'll make the system tool run in a container.

# mount requires CAP_SYS_ADMIN

OK, I suppose I should give some brief explanation on Linux kernel capabilities. An executing program used to inherit the privileges of the user running the program. If the user was `root` then the program had full permission to do anything. But kernel capabilities allows a non-root user to execute a program with a subset of those root permissions. [This LWN article](https://lwn.net/Articles/486306/) has been my general resource for kernel capabilities. You set the capabilities for the process you would be running before running it. They're basically flags.

One of these capabilities is called CAP_SYS_ADMIN. This gives a large amount of privileges to the process, so much so that the [manual](http://man7.org/linux/man-pages/man7/capabilities.7.html) calls it "as good as root". You will notice that `mount` is one of those operations you can only do with CAP_SYS_ADMIN. And this is the first problem I run into. Tern uses the `overlayfs` filesystem to mount the layers of a container image in order to collect information from it. It doesn't do it all the time. Only when the container image has more than one layer. However, a good majority of the container images out there have more than one layer, so it needs to do this. Why it needs to do this boils down to container builds in general. But anyway, containers are essentially kernel processes with capabilities and you can run a container with CAP_SYS_ADMIN. In Docker, you do that with `docker run --privileged`. If you run a container as privileged, you can run the `mount` command from within the container. And with this, I've already started paving the road to hell.

# You can't mount an overlay filesystem on top of an overlay filesystem

Right, now to mount image layers using `overlayfs`. This works on a generic Linux `ext3/4` filesystem (Side note: this is why Docker on Mac OSX and Windows runs within a VM). This does not work on another `overlay` filesystem. I did say a container is a running process, right? That process sees an overlay filesystem as its mount. That filesystem that you see when you run `docker run -it` is an overlay filesystem.

Well now, Tern will not be able to complete any of its overlay mount functions here. It needs to do this on some supported Linux filesystem. There's only one place to do this - on the host machine. How do you do this? By bind mounting a folder when running a container. A bind mount basically means that whatever you do in the mounted folder within the container gets reflected on the host machine. Oh boy...so run the container as privileged, and then bind mount a folder on the host machine to within the container. I've continued to pave the road to hell.

# I feel awful for the rest of the blog

This should raise some red flags about actually running this container out on a public cloud. DON'T DO IT!!!

I ended up putting a disclaimer in the [getting started with Docker guide](https://github.com/vmware/tern#getting-started-with-docker). Maybe it should have been a larger disclaimer.

Not like I haven't thought about ways out of my road to hell:

- Rootless containers. Now, when folks think about rootless containers, they mostly think about container runtime. That means there is an existing filesystem already mounted on disk and you can run a process with all the containery things without having to be a root user. This is possible now. It's also possible to mount a regular `ext` filesystem without being root. It requires some syscall gymnastics but I've managed to script it. It's not possible to mount an overlay filesystem this way (unless you're using Ubuntu, and that's pretty much what vendors do and say they're using rootless containers). This is not going upstream because of the general kernel brain trust decision that an overlay mount is too dangerous to allow a user process to do (although it seems fine to allow a user to mount a regular filesystem ¯\\_(ツ)_/¯).

- Using FUSE. I have to be honest - I haven't tried this and it's probably the only way out. It would basically mean building the Golang `overlayfs-fuse` implementation and using the binary. Sounds like a Makefile is in Tern's future. I have heard that FUSE is slow. I will have to do a comparison.

- Not use overlayfs at all. So like I said before, Tern is not a file scanner. The main reason is because, at this point, licenses are attached to projects consisting of a collection of files. Sometimes, a project has dependencies that have other licenses. Sometimes, you don't really know what license governs a single file. Sometimes you don't know how that file relates to some other files. In order for a lawyer (or a developer for that matter) to understand how the licenses work, they will need to understand the context in which they are used and distributed. You can't get this context at the file level. This is perhaps an issue that is overcome by proprietary databases, the curation of which is unknown to us. Tern is very transparent about where it gets its information from. It's up to you on what entity you want to trust, but for what it's worth, Tern could be hooked up to these entities. No pressure :)
