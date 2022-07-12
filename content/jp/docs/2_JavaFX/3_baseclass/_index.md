---
tags: ["JavaFX","FXML","gview"]
title: "共通クラス"
weight: 3
description: JavaFXウィンドウをKotlinで実装するために、今回のプログラムで作成した共通クラスを紹介します。
---

JavaFXプログラムを(kotlinで)記述するサンプルを紹介したのですが、実際にアプリを作るためにはもう少し整った構造が必要になります。

[オープンソースのフレームワーク](https://edvin.gitbooks.io/tornadofx-guide/content/)なども公開されていますが、対応するJavaやJavaFXのバージョンに制限があったりするので、もっと簡単な基本クラスを作成してみました。基本的な考え方は、
{{% alert theme="info" %}}FXMLファイルをクラスに関連付けて、ビュークラス+コントローラクラスという構成でウィンドウを実装できるようにする{{%/alert%}}
というものです。

### *BaseWindow*

FXMLファイルに関連付ける、ビューの基本クラスです。コンテナ単位での定義を想定しています。

```kotlin
import javafx.fxml.FXMLLoader
import javafx.scene.Parent

open class BaseWindow<Controller>(formPath: String)
		where Controller: BaseWindowCtrl {
	val root: Parent
	val controller: Controller

	init {
		val loader = FXMLLoader(javaClass.getResource(formPath))
		root = loader.load()
		controller = loader.getController() as Controller
	}
}
```

*formatPath*にはFXMLファイルのパスを、テンプレートパラメータの*Controller*には後述のコントローラクラスを指定します。FXMLファイルはコンテナ単位で作成する前提です。

インスタンスに対応してウィンドウのインスタンスが生成され、そのrootとコントローラが参照可能になります。

### *BaseWindowCtrl*

コントローラのベースクラスです。今のところ何も実装していませんが、共通の振る舞いを実装する場所として確保しておきます。

```kotlin
open class BaseWindowCtrl { }
```

### サンプルの実装

前の説明で作成したサンプルを、上の2つのクラスを使って実装してみました。FXMLファイル(*Hello.fxml*)の内容は、コントローラクラス名を除けば前回とまったく同じです。

```kotlin
<?xml version="1.0" encoding="UTF-8"?>

<?import javafx.scene.control.*?>
<?import javafx.scene.layout.*?>

<BorderPane xmlns="http://javafx.com/javafx/11.0.2"
            xmlns:fx="http://javafx.com/fxml/1"
            fx:controller="HelloWindowCtrl">
    <center>
        <Button fx:id="greetButton" />
    </center>
</BorderPane>
```

前回*Hello*クラスとして実装したコードを、以下のようなコントローラクラスとして実装します。FXMLファイルのコントローラにも、このクラス名を指定します。

```kotlin
import javafx.fxml.FXML
import javafx.scene.control.Button

class HelloWindowCtrl: BaseWindowCtrl() {
	@FXML private lateinit var greetButton: Button

	fun initialize() {
		greetButton.text = "Show"
		greetButton.setOnAction { _ -> println("Hello, World!") }
	}
}
```

ウィンドウクラスは*BaseWindow*を継承した簡単な実装になりました。

```kotlin
class HelloWindow: BaseWindow<HelloWindowCtrl>("/Hello.fxml") { }
```

アプリケーションクラスは次のように実装します。手順としては、
1. ウィンドウと*Scene*のインスタンスを生成する
1. メインウィンドウを表示する

```kotlin
import javafx.application.Application
import javafx.scene.Scene
import javafx.stage.Stage

class Sample: Application() {
	override fun start(primaryStage: Stage) {
		val hello = HelloWindow()
		val scene = Scene(hello.root, 300.0, 250.0)

		primaryStage.title = "Hello, World!"
		primaryStage.scene = scene
		primaryStage.show()
	}
}
```

実行結果は前回とまったく同じです。

![Hello, World](javafx_3.png)
