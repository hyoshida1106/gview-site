---
tags: ["JGit","kotlin"]
title: リポジトリモデルの作成
draft: false
weight: 1
description: >
    リポジトリモデルの実装について
---

Gitの情報は、リポジトリ単位で管理することにします。
最初にリポジトリを指定することにより、関連する情報が操作できるスタイルです。

今のところ、一度に操作できるリポジトリはひとつに限っていますが、
将来的には複数のリポジトリを同時に管理可能な方法も考えてみたいと思っています。

リポジトリクラスは、JGitの`Repository`をラップしたような、簡単な作りのクラスで、
関連する3つの情報を保持しています。

<dl>
<dt><em>workFiles</em></dt>
<dd>更新中のファイルやステージされたファイルなど、作業中のファイルを保持しています。</dd>
<dt><em>branches</em></dt>
<dd>ローカルおよびリモートのブランチ情報を保持しています。</dd>
<dt><em>commits</em></dt>
<dd>コミット情報を保持しています。</dd>
</dl>

JGitの`Repository`インスタンスは、コンストラクタパラメータとして与えらたものをプロパティとして公開します。また、インスタンス生成は後述のファクトリメソッドで実施するので、コンストラクタは`private`にしています。

```kotlin
class GvRepository private constructor(val jgitRepository: Repository) {
    val workFiles = GviewWorkFilesModel(this)
    val branches = GviewBranchListModel(this)
    val commits = GviewCommitListModel(this)
```

ファクトリメソッドも同じクラス内に、`companion object`のメソッドとして持つことにしました。リポジトリの作成には以下の3つの方法があるので、それぞれのメソッドを用意しています。

<dl>
<dt><em>init</em></dt>
<dd>指定されたディレクトリに新たなリポジトリを作成し、初期化します。</dd>
<dt><em>open</em></dt>
<dd>指定されたディレクトリをGitリポジトリとしてオープンします。</dd>
<dt><em>clone</em></dt>
<dd>指定されたURLから<em>clone</em>を実行し、同じく指定されたディレクトリに格納します。</dd>
</dl>

それぞれの処理自体は、JGitの`Git`クラスの静的メソッドを使えば簡単に実装することができます。

生成したインスタンスの参照もここに持つことにしました。変更時にコントローラに通知ができるように、
Observableなプロパティとして保持します。


以上の実装の結果、次のようなクラスが出来上がりました。

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
    val branches = GvBranchListModel(this)
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
