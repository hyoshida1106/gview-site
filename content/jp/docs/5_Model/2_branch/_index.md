---
tags: ["JGit","kotlin","branch"]
title: ブランチモデル
draft: false
weight: 2
description: >
    ブランチ情報を保持するモデルの実装について
---

リポジトリモデル内にローカルブランチとリモートブランチ、それぞれのリストを保持します。

### GvBranch

最初に、共通するブランチ情報を、抽象クラス *GvBranch* として実装しました。
インスタンスを保持するリストクラスである *GvBranchList* への参照と、情報を保持するJGitの *Ref* インスタンスへの参照を
コンストラクタのパラメータとして持つ、簡単なクラスです。

表示用の名称である *name* と、ブランチのパスを表す *path* は、abstractなメンバとして具象クラスで実装します。

```kotlin
package gview.model.branch
import org.eclipse.jgit.lib.Ref

abstract class GvBranch(val branchList: GvBranchList, val ref: Ref) {
    abstract val name: String
    abstract val path: String
}
```

### GvRemoteBranch

実装クラスは*GvLocalBranch*と*GvRemoteBranch*の２つで、それぞれローカルブランチとリモートブランチを表現します。  
*GvRemoreBranch*は対応するローカルブランチへの参照を持つだけの単純なクラスです。

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

メンバ*localBranch*がnullの場合は、チェックアウトされていないことを表します。

### GvLocalBranch

*GvLocalBranch*も同じようにリモートブランチへの参照を持っています。  
こちらのクラスは、カレントブランチであることを示すフラグ*isCurrentBranch*と、
ツリー表示の対象であることを示すプロパティ*selectedFlagProperty*を持っています。

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

リモートブランチとローカルブランチを保持するのは*GvBranchList*クラスです。  
まず、それぞれのリストと、現在選択されているブランチを示すフィールドを用意しました。

```kotlin
class GvBranchList(private val repository: GvRepository){
    val localBranchList = SimpleObjectProperty<List<GvLocalBranch>>()
    val remoteBranchList = SimpleObjectProperty<List<GvRemoteBranch>>()
    val currentBranch = SimpleStringProperty("")
```

このクラスは、コンストラクタのパラメータとして*GvRepository*を取得しています。  
JGitの*Repository*クラスのメソッドを利用して、リモートブランチとローカルブランチの名称を取得するメソッドを公開しています。
```kotlin
    fun remoteBranchDisplayName(name: String): String {
        return repository.jgitRepository.shortenRemoteBranchName(name)
    }
    fun localBranchDisplayName(name: String): String {
        return Repository.shortenRefName(name)
    }
```

ブランチリストへのインスタンスの設定はメソッド*update*で実行しています。
まず最初に、JGitのGitコマンド実行クラスである*Git*のインスタンスを取得します。
```kotlin
    private fun update() {
        val git = Git(repository.jgitRepository)
```

Gitインスタンスが取得できたら、最初にリモートブランチの一覧を取得します。 
リストモードを`REMOTE`にすることで、リモートブランチが取得できます。

取得したインスタンスをリストに格納すると同時に、後でローカルブランチと結び付けるためにマップ`remoteBranchMap`にも
設定しています。
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

続いてローカルブランチの一覧を取得します。リストモードを指定しないこと以外は、リモートブランチと同じです。  
ローカルブランチのインスタンスを取得したら、リモートブランチの取得時に作成したマップを使用して、
双方向の参照を張っておきます。
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

最後に、現在選択されているローカルブランチの情報を設定すれば、初期化は終了です。
この情報はリポジトリのインスタンスから取得します。
```kotlin
        currentBranch.value = repository.jgitRepository.branch
    }
```

*GvBranchList*には、ブランチインスタンスを操作するメソッドも実装しました。
当面は以下の3つを実装し、順次拡張することにします。
- リモートブランチのチェックアウト
- ローカルブランチのチェックアウト
- ローカルブランチの削除

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
