---
tags: ["JGit"]
title: "ライブラリについて"
draft: false
weight: 2
description: JGitのAPIに関する説明を簡単に行います。
---

JGitを使うにあたって、用意されているAPIとその他のクラスについて説明します。

### ドキュメント

JGitのサイトに[JGit/User Guide](https://wiki.eclipse.org/JGit/User_Guide)(英文)がありますが、
これを読んでも機能についてはほとんど分かりません。[JavaDoc](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/index.html)を調べるか、関連情報を検索することになると思います。

情報のまとまったサイトは、今のところJGit自体の[ドキュメントページ](https://www.eclipse.org/jgit/documentation/)以外に見つけることができていませんが、[StackOverflow](https://stackoverflow.com/)には関連する質問がけっこう上がっているようです。

### JGitの構成

資料の中にはっきりと書かれているわけではないのですが、ライブラリの構成を見れば、明らかに2つのレベルが存在していることが分かります。

ひとつは高次のAPIで、User Guideでは*Porcelain API*と表現されています。このAPIは簡単に言うとgitコマンドそのもので、コマンドで実行可能な機能はほとんど用意されています。

実行方法もコマンドとほぼ同じで、Gitコマンドに相当する[Gitクラス](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/api/Git.html)のメソッドを呼び出すだけです。それぞれのメソッドは、対応する[GitCommandクラス](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/api/GitCommand.html)のインスタンスを返すので、そのメソッドを使用して実際の操作を行います。

[紹介したサンプル]({{< relref "docs/3_JGit/1_setup#Sample" >}})でもクローンの実行に使用しましたが、gitコマンドを知っていれば、直感的な操作が可能で、プログラムが大幅に簡単になります。

もうひとつは、Gitクラスあるいはその下のGitCommandクラスが使用する下位クラスで、UserGuideで紹介されている[Ref](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/lib/Ref.html)、[RevWalk](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/revwalk/RevWalk.html)、[RevCommit](https://download.eclipse.org/jgit/site/5.12.0.202106070339-r/apidocs/org/eclipse/jgit/revwalk/RevCommit.html)といったクラスが相当します。

今回のプログラムでは、できる限りGitクラスを使用して、詳細な情報が必要な操作に限って下位クラスを使用することにしています。
具体的な使用方法は、プログラミングと並行して追記していく予定です。
なお、*Porcelain API*の"*Porcelain*"とは磁器、つまり洗面器や便器のようなもののことで、一般的なプログラミングを表現する”plumbing(配管)”に対して、もっと高レベルでユーザフレンドリな存在、というような意味なのだそうです。

[サンプル]({{< relref "docs/3_JGit/1_setup#Sample" >}})では、変数*git*がGitクラスのインスタンスです。これを使って、例えば次のように、リモートブランチの一覧を取得することなどが可能になります。

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