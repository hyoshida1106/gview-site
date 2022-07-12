---
tags: ["JGit"]
title: "JGit Library"
draft: false
weight: 2
description: >
  A brief description about JGit API.
---

This section describes the API and other classes that are available by using JGit.

### Documentation

The [JGit/User Guide](https://wiki.eclipse.org/JGit/User_Guide) is available on the JGit website, but It doesn't tell you much about the features.
You will probably have to look up [JavaDoc](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/index.html) or search for related information.

So far I haven't been able to find a site with all the information except for JGit's own [documentation page](https://www.eclipse.org/jgit/documentation/), but there seems to be a lot of related questions on [StackOverflow](https://stackoverflow.com/).

### JGit Structure

It is not stated in the document, but if you look at the structure of the library, you can clearly see that there are two levels.

The first is the high-level API, which is referred to as the *Porcelain API* in the User Guide. This API is actually the git command itself, and provides most of the functions that can be performed with that command.

The execution method is almost the same as the command, just call methods of the [Git class](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/api/Git.html).
Each method returns an instance of the corresponding [GitCommand class](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/api/GitCommand.html), so you can use that method to perform the given operation.

I used it to generate clones in [the sample we introduced]({{< relref "docs/3_JGit/1_setup#Sample" >}}). [If you know about the Git commands, you can use them intuitively, which makes the program much easier.

The other is subclasses used by the Git class or GitCommand classes, such as [Ref](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/lib/Ref.html), [RevWalk](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/revwalk/RevWalk.html), or [RevCommit](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/revwalk/RevCommit.html)described in the UserGuide.

In this program, we will use the Git class as much as possible, and use the lower classes only for operations that require detailed information.
The specific usage will be added in parallel with the programming.

In [sample]({{< relref "docs/3_JGit/1_setup#Sample" >}}), the variable *git* is an instance of the Git class. This can be used, for example, to get a list of remote branches, as shown below:

```kotlin
	val git = Git.cloneRepository()
		.setURI(remotePath)
		.setDirectory(File(localPath))
		.setBare(false)
		.call()

	git.branchList()
		.setListMode(ListBranchCommand.ListMode.REMOTE)
		.call()
		.forEach { println(it.name) }
```