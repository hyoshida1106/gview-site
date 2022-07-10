---
tags: ["JavaFX","kotlin","FXML","Window"]
title: "Window基本クラス"
date: 2022-01-30
description: >
    JavaFXウィンドウをKotlinで実装するために、共通的に利用可能なクラスを作成しました。<br/>
    FXMLで定義したフォームの処理を、Kotlinで簡単に実装することができます。
---

Gviewでは、[共通クラス]({{< relref "docs/2_JavaFX/3_baseclass" >}} "共通クラス")で説明したクラスをアレンジして、必要な機能を加えたウィンドウ共通クラスを用意しました。

### *GvBaseWindow*

ウィンドウの基本クラスです。内容としては [*BaseWindow*]({{< relref "docs/2_JavaFX/3_baseclass#BaseWindow" >}}) とほぼ同じですが、*ControlClass* の名称をパラメータに追加しました。

名称自体は処理中では使っていませんが、同じ名称を *StleClass* として加えておくことで、各コントロールクラスで独自のスタイル定義が可能なようにしてあります。

```kotlin
package gview.view.framework

import javafx.fxml.FXMLLoader
import javafx.scene.Parent

open class GvBaseWindow<Controller>(formPath: String, controlClass: String)
        where Controller: GvBaseWindowCtrl {

    val root: Parent
    val controller: Controller

    init {
        val loader = FXMLLoader(javaClass.getResource(formPath))
        root = loader.load()
        root.styleClass.add(controlClass)
        controller = loader.getController() as Controller
    }
}
```

### *GvBaseWindowControl*

コントロールクラスには、すべてのウィンドウインスタンスにイベントを送信するための仕組みを追加しました。

各インスタンスが内部クラス *WindowObserver* のインスタンスをシングルトンオブジェクトの持つに追加することで、すべてのインスタンスを呼び出しできるようにしています。

現時点で実装している機能は次の２つです。

*<dt>displayCompleted()</dt>*
<dd>
全体の描画が完了した時点で、*main* から呼び出しています。<br>
グリッドの幅を調整する場合など、全体のリソース配置が用意できた後に実行する必要のある処理を行います。
</dd>

*<dt>updateConfigInfo()</dt>*
<dd>
各ウィンドウの持っている情報の永続化を行うタイミングを通知します。
</dd>

```kotlin
package gview.view.framework

open class GvBaseWindowCtrl {
    // InnerClass - WindowObserver
    inner class WindowObserver {
        fun innerDisplayCompleted() { displayCompleted() }
        fun innerUpdateConfigInfo() { updateConfigInfo() }
    }
    // Class
    private val observer = WindowObserver()
    init {
        observers.add(observer)
    }
    fun finalize() {
        observers.remove(observer)
    }
    open fun displayCompleted() { }
    open fun updateConfigInfo() { }
    // Singleton
    companion object {
        val observers = mutableListOf<WindowObserver>()
        fun displayCompleted() { observers.forEach { it.innerDisplayCompleted() } }
        fun updateConfigInfo() { observers.forEach { it.innerUpdateConfigInfo() } }
    }
}
```

実際のクラスを実装するには、それぞれのクラスを継承した具象クラスを用意します。  
*MainWindow* の場合、次のようなコントロールクラスを実装しました。

```kotlin
class MainWindowCtrl: GvBaseWindowCtrl()
```

メインウィンドウはひとつなので、オブジェクトとして実装しました。
複数のインスタンスが必要な場合は、継承クラスとする必要があります。

```kotlin
package gview.view.main
import gview.view.framework.GvBaseWindow

object MainWindow: GvBaseWindow<MainWindowCtrl>("/view/MainView.fxml", "MainWindow")
```
