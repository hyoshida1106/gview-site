---
tags: ["JavaFX","FXML","gview"]
title: "Base Classes"
weight: 3
description:
  Here are the base classes created in this program.
---

I introduced a sample of writing a JavaFX program (in kotlin), but to actually create an application, we need a more well-structured one.

There are some [open-source frameworks](https://edvin.gitbooks.io/tornadofx-guide/content/) available, but they have restrictions on the supported Java or JavaFX versions, so I created simpler basic classes.
The basic idea is to associate an FXML file with a class so that a window can be implemented with a view class + controller class structure.

### *BaseWindow*

This is the base class for views that are associated with FXML files. It is assumed to be defined on a per-container basis:

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

*formatPath* is the path to the FXML file, and *Controller* in the template parameter is the controller class described later.
It is assumed that the FXML file is created in a container unit.

A window instance will be created corresponding to the class instance, and its root and controller will be available for reference.

### *BaseWindowCtrl*

This is the base class of the controller.
I have not implemented anything so far, but reserve it as a place to implement common behaviors.

```kotlin
open class BaseWindowCtrl { }
```

### Sample Implementation

We have implemented the sample created in the previous chapter using the two classes above.
The contents of the FXML file (*Hello.fxml*) are exactly the same as before except for the controller class name:

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

The code that was implemented last time as the *Hello* class is now implemented as a controller class as shown below.
This class name is also specified in the controller of the FXML file:

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

By inheriting from *BaseWindow*, the window class has been implemented in a very simple way.

```kotlin
class HelloWindow: BaseWindow<HelloWindowCtrl>("/Hello.fxml") { }
```

The application class is implemented in the following steps:
1. Create a window and an instance of *Scene*
1. Show the main window

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

The results are exactly the same as before:

![Hello, World](javafx_3.png)
