---
tags: ["JavaFX","kotlin","FXML","Window"]
title: "Basic Window Classes"
date: 2022-01-30
description: >
    This section describes the basic classes available for implementing JavaFX windows in Kotlin.<br/>
    You can easily implement the operations of the window defined in FXML in Kotlin.
---

For the gview implementation, I arranged the classes described in [Common Classes]({{< relref "docs/2_JavaFX/3_baseclass" >}} "Common Classes") and developed window common classes with necessary functions added.

---
### *GvBaseWindow*

The base class for windows.
The implementation is almost the same as [*BaseWindow*]({{< relref "docs/2_JavaFX/3_baseclass#BaseWindow" >}}), but the name of *ControlClass* has been added to the parameter.
The name itself is not used in the process, but it is added to *StleClass* so that each control class can have its own style definition.

{{< card-code header="**GvBaseWindow.kt**" lang="kotlin" >}}package gview.view.framework

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
{{< /card-code >}}<br/>

---
### *GvBaseWindowControl*

I added a mechanism to send events to all window instances.

Each instance owns an instance of the inner class *WindowObserver*, which is added to the singleton list so that all instances can be invoked.

So far, we have implemented the following two functions:

*<dt>displayCompleted()</dt>*
<dd>
It is called from *main* when the entire drawing is completed.<br>
Perform any processing that needs to be done after the overall resource layout is prepared, such as adjusting the grid width.
</dd>

*<dt>updateConfigInfo()</dt>*
<dd>
Notifies when to persist the information that each window has.
</dd>


{{< card-code header="**GvBaseWindowCtrl.kt**" lang="kotlin" >}}package gview.view.framework

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
{{< /card-code >}}<br/>

---
### Usage

To implement the actual window, prepare a concrete class that inherits from the respective class.
*In the case of MainWindow*, we have implemented the following control class:

```kotlin
class MainWindowCtrl: GvBaseWindowCtrl()
```

Since there is only one main window, we implemented it as an object.
If you need multiple instances, you could implement it as a factory class.

```kotlin
package gview.view.main
import gview.view.framework.GvBaseWindow

object MainWindow: GvBaseWindow<MainWindowCtrl>("/view/MainView.fxml", "MainWindow")
```
