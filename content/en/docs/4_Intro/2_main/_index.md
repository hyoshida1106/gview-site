---
tags: ["gview","JavaFX"]
title: Implementing Main Function
draft: false
weight: 2
description: >
    Create an entry for the application.
---

Now that the project is ready, the first step is to create an entry for the application.

### main() Function

Unlike Java, kotlin allows you to define functions that do not belong to a class.
It is the same as in Java that the application entry is `main()`, but there is no need to prepare a class and implement it as a static method.

The `main()` function of gview is a simple one, as shown below:
```kotlin
package gview
import javafx.application.Application

fun main(args: Array<String>) {
    Application.launch(GvApplication::class.java, *args)
}
```

The main function implements a simple process, which invokes `launch()` method of the `javafx.application.Application` class with `GvApplication`, the application class of gview, and the startup parameters. 

Since the first argument of `launch()` specifies a Java class, you need to describe it for that purpose. Other than that, there is no difference from developing in Java.

### Application Class

As mentioned above, I named the Application class of gview as `GvApplication`.
This class inherits from `javafx.application.Application`, which is JavaFX application class.
```kotlin
class GvApplication: Application()
```

##### *start()* Method

`GvApplication` class overrides the method `start()` to implement the process.
This method takes an instance of `Stage` as a parameter, so save it for later reference:
```kotlin
    private lateinit var mainStage: Stage
```

In the `start()` implementation, we first create a `Scene` instance and set it to `Stage`:
```kotlin
    override fun start(stage: Stage) {
        mainStage = stage
        mainStage.title = "GView"
        mainStage.scene = Scene(
                MainWindow.root,
                SystemModal.mainWidthProperty.value,
                SystemModal.mainHeightProperty.value)
```

The constructor of `Scene` has three parameters.  
* The first parameter, `MainWindow.root`, specifies the `Parent` instance of the main window of gview.
* The remaining two are the size of the window, so specify the size of the window at the last exit.

Now that we have a `Stage` instance, we can set up the necessary event handling:
```kotlin
        mainStage.onShown = EventHandler {
            GvBaseWindowCtrl.displayCompleted()
        }
        mainStage.onCloseRequest = EventHandler {
            confirmToQuit()
            it.consume()
        }
```

The first event, `onShown`, describes the process to be executed when the display is completed.
In this process, `GvBaseWindowsCtrl`, the window base class of gview, notifies each instance that the window drawing is completed.
Any processing that needs to be performed after drawing is completed in each window (such as grid sizing) is performed from this handler.

The `onCloseRequest` method is called when the window is about to be closed, and displays a dialog to confirm that the application is closed.

The following process is related to saving the window size specified in the `Scene` constructor:
```kotlin
        monitor.register(mainStage.scene)

        with(SystemModal) {
            mainHeightProperty.bind(mainStage.scene.heightProperty())
            mainWidthProperty.bind(mainStage.scene.widthProperty())
            maximumProperty.bind(mainStage.fullScreenProperty())
        }
```

The first step is to record the window size and create an instance of the idling monitoring process with the name `monitor`.
This instance monitors the operation of `mainStage` and updates the screen size information if there is no input for a certain period of time.

The next step is to bind the properties of the object named `SystemModal` to the size (height and width) and the maximize flag of the main window.
In the update process described above, this information is made persistent by serializing the object.

I would like to introduce idling monitoring and saving/restoring screen size information in another part of this article.

All that remains is to display the main window:
```kotlin
        mainStage.show()
```

With the addition of an exception handler for the entire process, the implementation of the `start()` method looks like the one below:
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

##### *confirmToQuit()* Method

This method is called when the main window is about to be closed, and confirm the termination of the application.  
If the input is "Yes", close the application:
```kotlin
    fun confirmToQuit() {
        val message = ResourceBundle.getBundle("Gview").getString("QuitConformation")
        if (ConfirmationDialog(ConfirmationType.YesNo, message).showDialog()) {
            exitProcess(0)
        }
    }
```
The `ConfirmationDialog` is a class that we implemented ourselves. 
It will be explained in another chapter.

##### *monitor* Instance

This is an instance of our own [idle timer]({{< relref "blog/topics/1_idletimer" >}} "idle timer") class, which monitors idle for one second:
```kotlin
    private val monitor = GvIdleTimer(1000) {
        GvBaseWindowCtrl.updateConfigInfo()
        SystemModal.saveToFile()
    }
```

It notifies each window of the information update (`updateConfigInfo()`), and serialize the configuration information (`SystemModal.saveToFile()`).
