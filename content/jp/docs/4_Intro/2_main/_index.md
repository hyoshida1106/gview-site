---
tags: ["gview","JavaFX"]
title: Mainの実装
draft: false
weight: 2
description: >
    アプリケーションのエントリを作成します。
---

プロジェクトの用意ができたので、アプリケーションのエントリから作成します。

### main()関数

Javaとは違い、kotlinではクラスに属さない関数を定義することが可能です。
アプリケーションのエントリが`main()`であることは同じですが、Javaのようにクラスを用意して、staticなメソッドとして実装する必要はありません。

gviewの`main()`関数は次のように、簡単なものになっています。

```kotlin
package gview
import javafx.application.Application

fun main(args: Array<String>) {
    Application.launch(GvApplication::class.java, *args)
}
```

`javafx.application.Application`クラスのstaticな`launch()`メソッドに、gviewのアプリケーション
クラスである`GvApplication`と起動パラメータを渡すだけの簡単な処理です。  
`launch()`の第1引数にはJavaクラスを指定する必要があるので、そのための記載をする以外は、Javaで開発する場合と違いはありません。

### Applicationクラス

前出の通り、gviewのApplicationクラスは`GvApplication`という名称にしました。
このクラスがJavaFXのアプリケーションクラスである`javafx.application.Application`を継承します。
```kotlin
class GvApplication: Application()
```
アプリケーションインスタンスを他クラスからアクセス可能にしたかったため、上のようなsingleton管理を行うことにしました。調べる限りでは、JavaFXでアプリケーションインスタンスを参照する方法はないようなので、このようにしたのですが、もっとスマートな方法があれば修正したいと思います。
```kotlin
    companion object {
        lateinit var instance : GvApplication
    }
    init {
        instance = this
    }
```

##### start()メソッド

`GvApplication`クラスでは、メソッド`start()`をオーバーライドして処理を実装します。
このメソッドには`Stage`のインスタンスが渡されるので、後で参照可能なように保存しています。
```kotlin
    private lateinit var mainStage: Stage
```
`start()`の実装では、最初に`Scene`インスタンスを生成して、`Stage`に設定します。
```kotlin
    override fun start(stage: Stage) {
        mainStage = stage
        mainStage.title = "GView"
        mainStage.scene = Scene(
                MainWindow.root,
                SystemModal.mainWidthProperty.value,
                SystemModal.mainHeightProperty.value)
```
`Scene`のコンストラクタには3つのパラメータを指定しています。  
* 最初の`MainWindow.root`はgviewのメインウィンドウの`Parent`インスタンスを指定します。
* 残る2つはウィンドウのサイズです。これには、前回終了時のウィンドウサイズが格納されています。

`Stage`インスタンスが入手できたので、必要なイベント処理を設定しておきます。
```kotlin
        mainStage.onShown = EventHandler {
            GvBaseWindowCtrl.displayCompleted()
        }
        mainStage.onCloseRequest = EventHandler {
            confirmToQuit()
            it.consume()
        }
```
最初のイベント`onShown`は、表示完了時に実行される処理を記載します。
ここでは、gviewのウィンドウ基本クラスである`GvBaseWindowsCtrl`から各インスタンスに表示終了を伝達しています。
各ウィンドウで描画完了後に実行する必要のある処理(グリッドのサイズ調整など)を、このハンドラから実行しています。

`onCloseRequest`はウィンドウを閉じようとする時に呼び出されるもので、ダイアログを表示して終了確認を行います。

続く処理は、最初の`Scene`コンストラクタで与えたウィンドウサイズの保存に関するものです。
```kotlin
        monitor.register(mainStage.scene)

        with(SystemModal) {
            mainHeightProperty.bind(mainStage.scene.heightProperty())
            mainWidthProperty.bind(mainStage.scene.widthProperty())
            maximumProperty.bind(mainStage.fullScreenProperty())
        }
```
最初の処理はウィンドウサイズを記録するためのもので、`monitor`という、アイドリング監視処理のインスタンスを生成しています。このインスタンスは`mainStage`の操作を監視して、一定時間以上入力のない場合に画面サイズ情報を更新する処理を実行しています。

以降の処理では、`SystemModal`というオブジェクトのプロパティをメインウィンドウのサイズ(高さ、幅)と最大指定にバインドしています。前述の更新処理では、このオブジェクトをシリアライズすることで、情報の保存を行っています。

アイドリング監視と画面サイズ情報の保存・復帰に関しては、別の部分で紹介したいと思います。

後はメインウィンドウの表示を行うだけです。
```kotlin
        mainStage.show()
```

処理全体の例外ハンドラを加えて、`start()`メソッドの実装は次のようになりました。
```kotlin
    override fun start(stage: Stage) {
        try {
            mainStage = stage
            mainStage.title = "GView"
            mainStage.scene = Scene(
                    MainWindow.root,
                    SystemModal.mainWidthProperty.value,
                    SystemModal.mainHeightProperty.value)

            mainStage.onShown = EventHandler {
                GvBaseWindowCtrl.displayCompleted()
            }

            mainStage.onCloseRequest = EventHandler {
                confirmToQuit()
                it.consume()
            }

            monitor.register(mainStage.scene)

            with(SystemModal) {
                mainHeightProperty.bind(mainStage.scene.heightProperty())
                mainWidthProperty.bind(mainStage.scene.widthProperty())
                maximumProperty.bind(mainStage.fullScreenProperty())
            }

            mainStage.show()

        } catch(e: java.lang.Exception) {
            e.printStackTrace()
            exitProcess(-1)
        }
    }
```

##### confirmToQuit()メソッド

メインウィンドウがクローズされようとしている時にコールされ、終了確認を行います。  
入力が"Yes"であれば、アプリケーションを終了します。
```kotlin
    fun confirmToQuit() {
        val message = ResourceBundle.getBundle("Gview").getString("QuitConformation")
        if (ConfirmationDialog(ConfirmationType.YesNo, message).showDialog()) {
            exitProcess(0)
        }
    }
```
クラス`ConfirmationDialog`は独自に定義したもので、これについても別途説明します。

##### monitorインスタンス

独自に実装した[アイドルタイマ]({{< relref "blog/topics/idletimer" >}} "アイドルタイマ")のインスタンスで、1秒間のアイドルを監視します。  
```kotlin
    private val monitor = GvIdleTimer(1000) {
        GvBaseWindowCtrl.updateConfigInfo()
        SystemModal.saveToFile()
    }
```
ここでは、各ウィンドウに情報更新を通知(`updateConfigInfo()`)した後、設定情報のシリアライズ(`SystemModal.saveToFile()`)を行っています。
