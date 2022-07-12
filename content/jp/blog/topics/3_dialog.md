---
tags: ["javaFX","kotlin","dialog"]
title: "カスタムダイアログ"
date: 2022-07-05
description: >
    JavaFXで独自のダイアログを実装するためのカスタムダイアログクラスをKotlinで開発しました。
---

JavaFXにはダイアログの実装クラスとして、[Alert](https://docs.oracle.com/javase/jp/8/javafx/api/javafx/scene/control/Alert.html)、[ChoiceDialog](https://docs.oracle.com/javase/jp/8/javafx/api/javafx/scene/control/ChoiceDialog.html)、[TextInputDialog](https://docs.oracle.com/javase/jp/8/javafx/api/javafx/scene/control/TextInputDialog.html)の3つがあります。
![JavaFXのダイアログ](/images/dialog.png)

これらのダイアログもそれなりに使えますが、もっと柔軟な定義をするために、独自のダイアログを[Dialog](https://docs.oracle.com/javase/jp/8/javafx/api/javafx/scene/control/Dialog.html)クラスを使って実装することにしました。

---
### GvDialogInterface

まず最初に、ダイアログの共通インターフェースを定義します。
これはJavaFXの標準ダイアログクラスを使用して実装したダイアログと、独自実装のダイアログを同じインターフェースで扱うために定義しています。

定義は簡単で、モーダル表示を実行するメソッドがひとつあるのみです。

{{< card-code header="**GvDialogInterface.kt**" lang="kotlin" >}}package gview.view.framework

interface GvDialogInterface<T> {
    fun showDialog(): T
}
{{< /card-code >}}

---
### GvCustomDialogCtrl

次に、独自ダイアログのコントロールクラスを抽象クラスとして実装します。
これも単純で、初期化処理を行うメソッドだけのクラスです。

{{< card-code header="**GvCustomDialogCtrl.kt**" lang="kotlin" >}}package gview.view.framework

abstract class GvCustomDialogCtrl {
    abstract fun initialize()
}
{{< /card-code >}}

---

### GvCustomDialog

この2つを使用して、独自ダイアログクラスを実装します。  

{{< card-code header="**GvCustomDialog.kt**" lang="kotlin" >}}package gview.view.framework

open class GvCustomDialog<Controller>(title: String, form: String, vararg buttons: ButtonType)
    : Dialog<ButtonType>(), GvDialogInterface<ButtonType?> where Controller : GvCustomDialogCtrl {

    val controller: Controller

    init {
        this.title = title
        dialogPane.buttonTypes.addAll(buttons)

        val loader = FXMLLoader(javaClass.getResource(form))
        dialogPane.content = loader.load()
        controller = loader.getController() as Controller

        dialogPane.stylesheets.add(javaClass.getResource("/Gview.css").toExternalForm())
        dialogPane.scene.window.onCloseRequest = EventHandler { it.consume() }
    }

    override fun showDialog(): ButtonType? {
        val result = super.showAndWait()
        return if (result.isPresent) result.get() else null
    }

    protected fun addButtonHandler(
        buttonType: ButtonType,
        disable: BooleanProperty?,
        handler: EventHandler<ActionEvent>? = null
    ) {
        val button = dialogPane.lookupButton(buttonType)
        if (button != null) {
            if (disable != null) {
                button.disableProperty().bind(disable)
            }
            if (handler != null) {
                button.addEventFilter(ActionEvent.ACTION, handler)
            }
        }
    }
}
{{< /card-code >}}

定義部分は次のようになっています。

```kotlin
open class GvCustomDialog<Controller>(title: String, form: String, vararg buttons: ButtonType)
    : Dialog<ButtonType>(), GvDialogInterface<ButtonType?> 
        where Controller : GvCustomDialogCtrl {
```

このクラスのコンストラクタには、3つのパラメータを指定します。

*title*
: ダイアログのタイトル文字列

*form*
: 画面フォーマットを定義したFXMLファイル名

*buttons*
: 表示するボタン、複数定義が可能

``controller``は、コントローラインスタンスを保持するプロパティです。

```kotlin
    val controller: Controller
```

初期化処理は次のような実装です。
最初に、パラメータ指定されたタイトル文字列とボタンを*Dialog*クラスの``title``と``dialogPane``に設定します。
```kotlin
    init() {
        this.title = title
        dialogPane.buttonTypes.addAll(buttons)
```

FXMLファイルをロードし、同じく``dialogPane``に設定するとともに、*FXMLLoader*からコントロールクラスのインスタンスを
取得して``controller``にセットします。コントロールクラスの指定はFXMLファイル内に記述します。
```kotlin
        val loader = FXMLLoader(javaClass.getResource(form))
        dialogPane.content = loader.load()
        controller = loader.getController() as Controller
```

``dialogPane``にスタイルシートを登録し、CSSでスタイル定義をできるようにします。  
最後の*EventHandler*設定は、[X]クリックなどでダイアログをクローズできなくするためのものです。
```kotlin
        dialogPane.stylesheets.add(javaClass.getResource("/Gview.css").toExternalForm())
        dialogPane.scene.window.onCloseRequest = EventHandler { it.consume() }
    }
```

次に、*GvDialogInterface*で定義されているメソッドの実装を行います。  
*Dialog*クラスのメソッド``showAndWait``をコールした上で、結果が有効であれば*ButtonType*を取得して返しています。
```kotlin
    override fun showDialog(): ButtonType? {
        val result = super.showAndWait()
        return if (result.isPresent) result.get() else null
    }
```

最後に、ボタン押下時のイベント処理を登録するためのメソッドを実装しておきます。
このメソッドは、独自ダイアログ実装クラスから呼び出される前提です。
```kotlin
    protected fun addButtonHandler(
        buttonType: ButtonType,
        disable: BooleanProperty?,
        handler: EventHandler<ActionEvent>? = null
    ) {
        val button = dialogPane.lookupButton(buttonType)
        if (button != null) {
            if (disable != null) {
                button.disableProperty().bind(disable)
            }
            if (handler != null) {
                button.addEventFilter(ActionEvent.ACTION, handler)
            }
        }
    }
```

このメソッドには、以下の3つのパラメータを指定可能です。

*buttonType*
: 処理を設定するボタンを*ButtonType*で指定します。
コンストラクタの``buttons``で指定した中のひとつである必要があります。

*disable*
: ボタンの有効/無効を設定するプロパティを指定します(省略可)。
``null``以外が指定された場合、そのプロパティの値を設定することで、ボタンを有効/無効に
することができます。

*handler*
: ``null``ではない*EventHandler*が指定された場合、ボタン押下時に呼び出されます。

以上でカスタムダイアログの実装は完了です。

---

### 使用例

ここではカスタムダイアログクラスを使用した実装例を紹介したいと思います。  
次のダイアログは、Gviewでローカルブランチを生成する時に表示するダイアログです。

![実装例](/images/customdialog.png)

ダイアログはラベルとテキストボックス、チェックボックスで構成されていて、
[OK][キャンセル]の２つのボタンで閉じることができます。  
最初に作成するのは、表示を定義するFXMLファイルです。

{{< card-code header="**BranchNameDialog.fxml**" lang="XML" >}}<?xml version="1.0" encoding="UTF-8"?>

<?import javafx.geometry.Insets?>
<?import javafx.scene.layout.ColumnConstraints?>
<?import javafx.scene.layout.GridPane?>
<?import javafx.scene.control.Label?>
<?import javafx.scene.control.TextField?>

<?import javafx.scene.control.CheckBox?>
<GridPane hgap="20.0" maxHeight="-Infinity" maxWidth="-Infinity" minHeight="-Infinity" minWidth="-Infinity"
          fx:id="pane" fx:controller="gview.view.dialog.BranchNameDialogCtrl"
          vgap="12.0" xmlns:fx="http://javafx.com/fxml/1" xmlns="http://javafx.com/javafx/11.0.1">
    <columnConstraints>
        <ColumnConstraints hgrow="ALWAYS"/>
        <ColumnConstraints hgrow="ALWAYS"/>
    </columnConstraints>
    <padding>
        <Insets bottom="12.0" left="20.0" right="20.0" top="20.0"/>
    </padding>
    <Label text="ブランチの名称:" GridPane.halignment="RIGHT"/>
    <TextField fx:id="branchName" prefColumnCount="20" GridPane.columnIndex="1"/>
    <CheckBox fx:id="checkout" text="作成したブランチをチェックアウトする" 
              GridPane.rowIndex="1" GridPane.columnSpan="2"/>
</GridPane>
{{< /card-code >}}<br/>

*GridPane*を使って*Label*、*TextField*、2行目に*CheckBox*を配置しています。
コントロールクラスの定義もここで行います。

次に、FXMLで定義した名称のコントロールクラスを、基本クラス*GvCustomDialogCtrl*を継承して実装します。

{{< card-code header="**BranchNameDialogCtrl.kt**" lang="kotlin" >}}package gview.view.dialog

import gview.view.framework.GvCustomDialogCtrl
import javafx.beans.property.SimpleBooleanProperty
import javafx.fxml.FXML
import javafx.scene.control.CheckBox
import javafx.scene.control.TextField

class BranchNameDialogCtrl : GvCustomDialogCtrl() {
    @FXML private lateinit var branchName: TextField
    @FXML private lateinit var checkout: CheckBox

    val btnOkDisable = SimpleBooleanProperty(true)

    val newBranchName get() = branchName.text.trim()
    val checkoutFlag: Boolean get() = checkout.isSelected

    override fun initialize() {
        checkout.isSelected = true
        branchName.text = ""
        branchName.lengthProperty().addListener { _ -> 
            btnOkDisable.value = branchName.text.isBlank() }
    }
}
{{< /card-code >}}<br/>

最初に、FXMLで定義したコンポーネント*TextField*と*Checkbox*のプロパティインスタンスを宣言します。
```kotlin
class BranchNameDialogCtrl : GvCustomDialogCtrl() {
    @FXML private lateinit var branchName: TextField
    @FXML private lateinit var checkout: CheckBox
```

[OK]ボタンの有効/無効を切り替える*Property*インスタンスを宣言します。
ボタンとの結合は、後述の*BranchNameDialog*クラスで行っています。

その他、*TextField*と*Checkbox*の設定内容を参照するためのプロパティも用意しておきます。
```kotlin
    val btnOkDisable = SimpleBooleanProperty(true)

    val newBranchName get() = branchName.text.trim()
    val checkoutFlag: Boolean get() = checkout.isSelected
```

抽象メソッド``initialize()``を実装します。  
入力フィールドに初期値を設定した上で、*TextField*の長さに対するイベントハンドラを定義します。
イベントハンドラでは、テキスト長が０の場合に[OK]ボタンを無効にするようにしています。
```kotlin
    override fun initialize() {
        checkout.isSelected = true
        branchName.text = ""
        branchName.lengthProperty().addListener { _ -> 
            btnOkDisable.value = branchName.text.isBlank() }
    }
```

*BranchNameDialog*クラスでは、タイトル文字列とFXMLファイルのパス、表示するボタン([OK]と[キャンセル])を
指定しています。  
初期化処理では、先程の[OK]ボタンを制御するプロパティとボタンとの結合を行っています。
``controller``には、継承する*GvCustomDialog<BranchNameDialogCtrl>*クラスの初期化の中で、
コントローラインスタンスに対する参照が事前に設定されています。
{{< card-code header="**BranchNameDialog.kt**" lang="kotlin" >}}package gview.view.dialog

import gview.view.framework.GvCustomDialog
import javafx.scene.control.ButtonType

class BranchNameDialog() : GvCustomDialog<BranchNameDialogCtrl>(
    "作成するブランチの名前を指定してください",
    "/dialog/BranchNameDialog.fxml",
    ButtonType.OK, ButtonType.CANCEL
) {
    init {
        addButtonHandler(ButtonType.OK, controller.btnOkDisable)
    }
}
{{< /card-code >}}<br/>

以上で実装は終了です。  
使用時は*BranchNameDialog*のインスタンスを生成して``showDialog()``を呼び出せば、
選択されたボタンが取得できます。

```kotlin
    val dialog = BranchNameDialog()
    if(dialog.showDialog() == ButtonType.OK) {
        val branchName: String = dialog.controller.newBranchName
        val chekoutFlag: Boolean = dialog.controller.checkoutFlag
        // 処理を行う
    } else {
        // キャンセルされた
    }
```
