---
tags: ["javaFX","kotlin","dialog"]
title: "Custom Dialog in JavaFX"
date: 2022-07-05
description: >
    Implementing JavaFX custom dialog class in kotlin.
---

JavaFX has three dialog implementation classes: [Alert](https://docs.oracle.com/javase/jp/8/javafx/api/javafx/scene/control/Alert.html), [ChoiceDialog](https://docs.oracle.com/javase/jp/8/javafx/api/javafx/scene/control/ChoiceDialog.html), and [TextInputDialog](https://docs.oracle.com/javase/jp/8/javafx/api/javafx/scene/control/TextInputDialog.html).
![JavaFX Dialogs](/images/dialog.png)

These dialogs have some use,  but for more flexibility, I decided to implement my own dialog framework with the JavaFX [Dialog](https://docs.oracle.com/javase/jp/8/javafx/api/javafx/scene/control/Dialog.html) class.

---
### Dialog Common Interface

The first step was to define a common interface for the dialog.
This is defined to handle dialogs implemented using the standard JavaFX dialog classes, and dialogs with our own framework, in the same interface.
It is very simple interface with only a method to execute the modal display.

{{< card-code header="**GvDialogInterface.kt**" lang="kotlin" >}}package gview.view.framework

interface GvDialogInterface<T> {
    fun showDialog(): T
}
{{< /card-code >}}

---
### Abstract Dialog Control Class

Next, I implemented custom dialog control class as an abstract class. 
This is another simple class with only a method to perform initialization.

{{< card-code header="**GvCustomDialogCtrl.kt**" lang="kotlin" >}}package gview.view.framework

abstract class GvCustomDialogCtrl {
    abstract fun initialize()
}
{{< /card-code >}}

---

### Custom Dialog Class

With these two, I wrote my custom dialog class.

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

The declaration part of the class is as follows:

```kotlin
open class GvCustomDialog<Controller>(title: String, form: String, vararg buttons: ButtonType)
    : Dialog<ButtonType>(), GvDialogInterface<ButtonType?> 
        where Controller : GvCustomDialogCtrl {
```

The constructor has three parameters.

*title*
: Title string of dialog

*form*
: FXML file name defining the screen format

*buttons*
: Buttons to be displayed, multiple definitions possible

``controller`` is a property that holds a controller instance.

```kotlin
    val controller: Controller
```

The initialization process is implemented as follows:  
First, we set the title string and button(s) to ``title`` and ``dialogPane`` of the Dialog class.
```kotlin
    init() {
        this.title = title
        dialogPane.buttonTypes.addAll(buttons)
```

Load the FXML file, set it to ``dialogPane`` as well, and get an instance of the control class from FXMLLoader and set it to ``controller``. 
The control class is specified in the FXML file.
```kotlin
        val loader = FXMLLoader(javaClass.getResource(form))
        dialogPane.content = loader.load()
        controller = loader.getController() as Controller
```

Register a style sheet in the dialogPane so that you can define the style with CSS.
And finally, we set up an EventHandler to prevent the dialog from being closed by clicking [X].
```kotlin
        dialogPane.stylesheets.add(javaClass.getResource("/Gview.css").toExternalForm())
        dialogPane.scene.window.onCloseRequest = EventHandler { it.consume() }
    }
```

The next step is to implement the methods defined in GvDialogInterface.
The following implementation calls the method ``showAndWait`` of the *Dialog* class,
 and then returns the *ButtonType* if the result is valid.
```kotlin
    override fun showDialog(): ButtonType? {
        val result = super.showAndWait()
        return if (result.isPresent) result.get() else null
    }
```

The last thing to do is to implement a method to handle events when a button is pressed.
This method is assumed to be called from a dialog implementation class that extends this class.
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

This method has three parameters.

*buttonType*
: Specify the target button by *ButtonType*.
It must be one of those specified in the ``buttons`` parameter of the constructor.

*disable*
: Specifies the property that enables/disables the button (optional).
If not ``null``, you can enable/disable the button by setting the value of this property.

*handler*
: If an *EventHandler* that is not ``null`` is specified, it will be called when the button is pressed.

Now, the implementation of the custom dialog is complete.

---

### Example

Here I would like to present an example implementation using this custom dialog class.  
The example we use is the dialog that is displayed when creating a local branch in Gview.

![exapmple](/images/customdialog.png)

This dialog has labels, text boxes, and check boxes, and can be closed with the "OK" or "Cancel" buttons.   
The first step is to write the FXML file that defines the components and their placement.

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

In the file above, a *GridPane* is used to place a *Label* and a *TextField* on the first line, and a *CheckBox* on the second line. 
The control class is also defined in this file.

Next, we implement a control class with the name specified in the FXML file, inheriting from the base class *GvCustomDialogCtrl*.

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

First, declare property instances of the *TextField* and *Checkbox* components defined in FXML.
```kotlin
class BranchNameDialogCtrl : GvCustomDialogCtrl() {
    @FXML private lateinit var branchName: TextField
    @FXML private lateinit var checkout: CheckBox
```

And declare a *Property* instance that enables/disables the [OK] button.
The binding to the button is done in the *BranchNameDialog* class described later.

Properties for referencing *TextField* and *Checkbox* values should also be provided.
```kotlin
    val btnOkDisable = SimpleBooleanProperty(true)

    val newBranchName get() = branchName.text.trim()
    val checkoutFlag: Boolean get() = checkout.isSelected
```

Next, we write an implementation of the abstract method ``initialize()``.    
Within this implementation, we set initial values for input fields and define event handlers for the length of the *TextField*.
This event handler disables the [OK] button if the text length is 0.
```kotlin
    override fun initialize() {
        checkout.isSelected = true
        branchName.text = ""
        branchName.lengthProperty().addListener { _ -> 
            btnOkDisable.value = branchName.text.isBlank() }
    }
```

In the *BranchNameDialog* class, the title string, the path to the FXML file, and the buttons to display ([OK] and [Cancel]) are specified to the constructor of the super class. 
In the initialization method, the button is combined with the property that controls the [OK] button described earlier.
The ``controller`` is pre-set with a reference to the controller instance in the initialization of the inheriting *GvCustomDialog<BranchNameDialogCtrl>* class.
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

Implementation has been completed.  
When using this dialog, create an instance of *BranchNameDialog* and call ``showDialog()`` and the selected button will be returned.

```kotlin
    val dialog = BranchNameDialog()
    if(dialog.showDialog() == ButtonType.OK) {
        val branchName: String = dialog.controller.newBranchName
        val chekoutFlag: Boolean = dialog.controller.checkoutFlag
        // Perform any process.
    } else {
        // Canceled.
    }
```
