---
tags: ["JGit","kotlin","branch"]
title: Branch Model
draft: false
weight: 2
description: >
    Implementation of the branch model
---

We keep lists of local and remote branches, each in the repository model.

### GvBranch

I implemented common branch information as an abstract class called *GvBranch*.

It is a simple class that has a reference to *GvBranchList*, a list class that holds instances of *GvBranchList*, and a reference to *Ref* instances of JGit that hold information, as constructor parameters.

The *name*, which is the name for display, and the *path*, which represents the path of the branch, are abstract members, which will be implemented in concrete classes.

```kotlin
package gview.model.branch
import org.eclipse.jgit.lib.Ref

abstract class GvBranch(val branchList: GvBranchList, val ref: Ref) {
    abstract val name: String
    abstract val path: String
}
```

### GvRemoteBranch

There are two implementation classes, *GvLocalBranch* and *GvRemoteBranch*, representing local and remote branches, respectively.  
*GvRemoreBranch* is a simple class with a reference to the corresponding local branch.

```kotlin
package gview.model.branch
import org.eclipse.jgit.lib.Ref
import java.lang.ref.WeakReference

class GvRemoteBranch(branchList: GvBranchList, ref: Ref): GvBranch(branchList, ref) {
    override val name: String = branchList.remoteBranchDisplayName(ref.name)
    override val path: String = ref.name
    var localBranch = WeakReference<GvLocalBranch>(null)
}
```
If the member *localBranch* is null, it means it is not checked out.

### GvLocalBranch

*GvLocalBranch* likewise has a reference to a remote branch.  

This class has *isCurrentBranch* flag to indicate to be the current branch
and *selectedFlagProperty* property to indicate to be the target of the tree view.

```kotlin
package gview.model.branch
import javafx.beans.property.SimpleBooleanProperty
import org.eclipse.jgit.lib.Ref
import java.lang.ref.WeakReference

class GvLocalBranch(branchList: GvBranchList, ref: Ref): GvBranch(branchList, ref) {
    override val name: String = branchList.localBranchDisplayName(ref.name)
    override val path: String = ref.name
    var remoteBranch = WeakReference<GvRemoteBranch>(null)
    val isCurrentBranch: Boolean get() = branchList.currentBranch.value == name
    val selectedFlagProperty = SimpleBooleanProperty(true)
}
```

### GvBranchList

The *GvBranchList* class holds remote and local branches.  
First, we have fields for two lists and the currently selected branch.

```kotlin
class GvBranchList(private val repository: GvRepository){
    val localBranchList = SimpleObjectProperty<List<GvLocalBranch>>()
    val remoteBranchList = SimpleObjectProperty<List<GvRemoteBranch>>()
    val currentBranch = SimpleStringProperty("")
```

This class gets *GvRepository* as a constructor parameter.  
Using the methods of JGit's *Repository* class, we expose the methods to get the names of remote and local branches.

```kotlin
    fun remoteBranchDisplayName(name: String): String {
        return repository.jgitRepository.shortenRemoteBranchName(name)
    }
    fun localBranchDisplayName(name: String): String {
        return Repository.shortenRefName(name)
    }
```

*Update* method sets up the branch lists.  
First, we get an instance of *Git*, JGit's Git command execution class.

```kotlin
    private fun update() {
        val git = Git(repository.jgitRepository)
```

Once the Git instance is obtained, the first step is to get a list of remote branches.  
Remote branches can be obtained by setting the list mode to `REMOTE`.

The retrieved instances are stored in a list and also set in a map `remoteBranchMap`, 
which is used later to be associated with local branches.

```kotlin
        val remoteBranches = mutableListOf<GvRemoteBranch>()
        val remoteBranchMap = mutableMapOf<String, GvRemoteBranch>()
        git.branchList()
            .setListMode(ListBranchCommand.ListMode.REMOTE)
            .call()
            .forEach {
                val remoteBranch = GvRemoteBranch(this, it)
                remoteBranches.add(remoteBranch)
                remoteBranchMap[remoteBranch.path] = remoteBranch
            }
        remoteBranchList.value = remoteBranches
```

Next, we get a list of local branches, which is same as for remote branches,
except that list mode is not set.  
Once obtained instances of the local branch,  we put up two-way references (remote-local)
with the map we created when obtaining the remote branch.

```kotlin
        val localBranches = mutableListOf<GvLocalBranch>()
        git.branchList()
            .call()
            .forEach {
                val localBranch = GvLocalBranch(this, it)
                localBranches.add(localBranch)
                val trackingStatus = BranchTrackingStatus.of(
                    repository.jgitRepository, localBranch.path)
                if (trackingStatus != null) {
                    val remoteBranch = remoteBranchMap[trackingStatus.remoteTrackingBranch]
                    if (remoteBranch != null) {
                        localBranch.remoteBranch = WeakReference(remoteBranch)
                        remoteBranch.localBranch = WeakReference(localBranch)
                    }
                }
            }
        localBranchList.value = localBranches
```

Finally,  we set the information for the currently selected local branch, and complete the initialization.
This information is obtained from the repository instance.

```kotlin
        currentBranch.value = repository.jgitRepository.branch
    }
```

I also implemented some methods for manipulating branch instances in *GvBranchList*.
For the time being, the following three are implemented and will gradually extend.

- Check out remote branches
- Check out local branches
- Delete local branch

```kotlin
    fun checkoutRemoteBranch(model: GvRemoteBranch) {
        Git(repository.jgitRepository)
                .checkout()
                .setName(model.name)
                .setStartPoint(model.path)
                .setUpstreamMode(CreateBranchCommand.SetupUpstreamMode.TRACK)
                .setCreateBranch(true)
                .call()
        Platform.runLater { update() }
    }

    fun checkoutLocalBranch(model: GvLocalBranch) {
        Git(repository.jgitRepository)
                .checkout()
                .setName(model.name)
                .call()
        Platform.runLater { update() }
    }

    fun removeLocalBranch(model: GvLocalBranch, force: Boolean) {
        Git(repository.jgitRepository)
                .branchDelete()
                .setBranchNames(model.name)
                .setForce(force)
                .call()
        Platform.runLater { update() }
    }
```
