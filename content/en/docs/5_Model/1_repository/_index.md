---
tags: ["JGit","kotlin","repository"]
title: Creating Repository Model
draft: false
weight: 1
description: >
    Implementation of the repository model
---

Git information is managed on a per-repository basis.
This style allows us to manipulate related information by specifying the repository.

At the moment, only one repository can be operated at a time, but in the future I would like to consider a way to manage multiple repositories simultaneously.

The repository class is a simple class that wraps `Repository` of JGit, and holds three related pieces of information:
<dl>
<dt><em>workFiles</em></dt>
<dd>Holds files that are being worked on, such as being updated or staged.</dd>
<dt><em>branches</em></dt>
<dd>Contains local and remote branch information.</dd>
<dt><em>commits</em></dt>
<dd>Holds commit information.</dd>
</dl>

It exposes the `Repository` instance of JGit given as a constructor parameter as a property.
The constructor is set to `private` because the instance is created using the factory method described below.

```kotlin
class GvRepository private constructor(val jgitRepository: Repository) {
    val workFiles = GviewWorkFilesModel(this)
    val branches = GviewBranchListModel(this)
    val commits = GviewCommitListModel(this)
```

I also decided to have the factory method as a method of the `companion object` in the same class.
There are three ways to create a repository, so we have methods for each of them.

<dl>
<dt><em>init</em></dt>
<dd>Creates and initializes a new repository in the specified directory.</dd>
<dt><em>open</em></dt>
<dd>Opens the specified directory as a Git repository.</dd>
<dt><em>clone</em></dt>
<dd>Execute <em>clone</em> from the specified URL, and store it in the specified directory as well.</dd>
</dl>

Each of these processes can be easily implemented using static methods of the JGit `Git` class.

I also have a reference to the generated instance here.
I keep it as an Observable property so that the controller can be notified when the instance is regenerated.

As a result of the above implementation, we have the following class:

{{< card-code header="**GvRepository.kt**" lang="kotlin" >}}package gview.model

import gview.model.branch.GviBranchListModel
import gview.model.commit.GviCommitListModel
import gview.model.workfile.GvwWorkFilesModel
import javafx.beans.property.SimpleObjectProperty
import org.eclipse.jgit.api.Git
import org.eclipse.jgit.lib.Repository
import java.io.File

class GvRepository private constructor(val jgitRepository: Repository) {

    val workFiles = GvWorkFilesModel(this)
    val branches = GvBranchList(this)
    val commits = GvCommitListModel(this)

    companion object {
        val currentRepositoryProperty = SimpleObjectProperty<GvRepository>()
        val currentRepository: GvRepository? get() = currentRepositoryProperty.value

        fun init(directoryPath: String, isBare: Boolean = false) {
            currentRepositoryProperty.set(GvRepository(
                Git.init()
                    .setBare(isBare)
                    .setDirectory(File(directoryPath))
                    .setGitDir(File(directoryPath, ".git"))
                    .call()
                    .repository
            ))
        }

        fun open(directoryPath: String) {
            currentRepositoryProperty.set(GvRepository(
                Git.open(File(directoryPath))
                    .repository
            ))
        }

        fun clone(directoryPath: String, remoteUrl: String, isBare: Boolean = false) {
            currentRepositoryProperty.set(GvRepository(
                Git.cloneRepository()
                    .setURI(remoteUrl)
                    .setDirectory(File(directoryPath))
                    .setBare(isBare)
                    .call()
                    .repository
            ))
        }
    }
}
{{< /card-code >}}
