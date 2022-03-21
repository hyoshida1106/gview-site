---
tags: ["gview","JavaFX"]
title: Implementing MainWindow
draft: false
weight: 1
description: >
    Describes the implementation of MainWindow.
---

This section describes the implementation of MainWindow.

Although I use *JavaFX* for the GUI, I have created a simple common class for this application.
The implementation details of the common class are described in [Window Basic Class]({{< relref "blog/topics/2_windowclass" >}}).

### Basic Structure and FXML

The main window of gview is structured as shown below:
<img src="mainScreen.png" />

The screen consists of three panes: `menuBar`, `mainSplit`, and `statusBar`. 
`mainSplit` is a split window with three more panes: `branchList`, `commitList`, and `commitInfo`.

This configuration is represented by the following FXML file.
*MaskerPane* is used to display the running status (spinning icon). 
The details will be explained separately.

{{< card-code header="**MainView.fxml**" lang="xml" >}}<?xml version="1.0" encoding="UTF-8"?>

<?import javafx.scene.control.*?>
<?import javafx.scene.layout.*?>

<?import org.controlsfx.control.MaskerPane?>
<StackPane
        xmlns="http://javafx.com/javafx/10.0.2" xmlns:fx="http://javafx.com/fxml/1"
        stylesheets="@/Gview.css" fx:controller="gview.view.main.MainWindowCtrl">
    <BorderPane>
        <top>
            <AnchorPane fx:id="menuBar"/>
        </top>
        <center>
            <SplitPane fx:id="mainSplit" dividerPositions="0.2, 0.5" BorderPane.alignment="CENTER">
                <AnchorPane fx:id="branchList" SplitPane.resizableWithParent="false"/>
                <AnchorPane fx:id="commitList" SplitPane.resizableWithParent="false"/>
                <AnchorPane fx:id="commitInfo"/>
            </SplitPane>
        </center>
        <bottom>
            <AnchorPane fx:id="statusBar"/>
        </bottom>
    </BorderPane>
    <MaskerPane fx:id="masker" visible="false"/>
</StackPane>
{{< /card-code >}}

### Window Class

The `MainWindow` class, which is the view class of the main window, can be short by inheriting from the base class:
{{< card-code header="**MainWindow.kt**" lang="kotlin" >}}import gview.view.framework.GvBaseWindow

object MainWindow: GvBaseWindow<MainWindowCtrl>("/view/MainView.fxml", "MainWindow")
{{< /card-code >}}

### Control Class

The control class also inherits from the base class.
The first step is to define the variables corresponding to each Pane defined in the FXML file:
```kotlin
class MainWindowCtrl: GvBaseWindowCtrl() {
    @FXML private lateinit var mainSplit: SplitPane
    @FXML private lateinit var menuBar: AnchorPane
    @FXML private lateinit var branchList: AnchorPane
    @FXML private lateinit var commitList: AnchorPane
    @FXML private lateinit var commitInfo: AnchorPane
    @FXML private lateinit var statusBar: AnchorPane
    @FXML private lateinit var masker: MaskerPane
```

The `initialize()` method does two things.

First, it sets `mainSprit`. After setting the initial value, add an event definition to get the value when the display ratio is changed.
The `SystemModal` object is a data storage class that supports serialization/deserialization to a file, retaining the display ratio at the end of the previous time and persisting it to a file when it is changed:
```kotlin
    fun initialize() {
        mainSplit.setDividerPositions(
            SystemModal.mainSplitPos[0], SystemModal.mainSplitPos[1])
        mainSplit.dividers[0].positionProperty().addListener { _, _, value
            -> SystemModal.mainSplitPos[0] = value.toDouble() }
        mainSplit.dividers[1].positionProperty().addListener { _, _, value
            -> SystemModal.mainSplitPos[1] = value.toDouble() }

        branchList.children.add(BranchList.root)
        commitList.children.add(CommitList.root)
        commitInfo.children.add(CommitInfo.root)
        menuBar.children.add(MenuBar.root)
        statusBar.children.add(StatusBar.root)
    }
```

The second half is setting up the windows for each Pane.
Each window, such as `BranchList`, `CommitList`, etc., is an object with the same structure as `MainWindow`, so you can easily configure the window by simply referring to the *Parent* instance and setting it.

The actual processing is implemented in each Pane, so there is very little processing in `MainWindowCtrl` itself.
The only method, `runTask()`, executes the specified process behind a "running" display, specifically a processing screen display with a *WAIT* icon and `masker`.
It is used in the repository open process at program startup:
```kotlin
    fun runTask(proc: () -> Unit) {
        val scene = MainWindow.root.scene
        scene.cursor = Cursor.WAIT
        val task = object: Task<Unit>() {
            override fun call() {
                try {
                    proc()
                } catch(e: Exception) {
                    Platform.runLater { ErrorDialog(e).showDialog() }
                }
                finally { Platform.runLater { scene.cursor = Cursor.DEFAULT } }
            }
        }
        masker.visibleProperty().bind(task.runningProperty())
        Thread(task).start()
    }
```

That's all there is to the `MainWindowCtrl` implementation. 
Overall, it looks like the following:
{{< card-code header="**MainWindowCtrl.kt**" lang="kotlin" >}}package gview.view.main

import gview.conf.SystemModal
import gview.view.branchlist.BranchList
import gview.view.commitinfo.CommitInfo
import gview.view.commitlist.CommitList
import gview.view.dialog.ErrorDialog
import gview.view.framework.GvBaseWindowCtrl
import javafx.application.Platform
import javafx.concurrent.Task
import javafx.fxml.FXML
import javafx.scene.Cursor
import javafx.scene.control.SplitPane
import javafx.scene.layout.AnchorPane
import org.controlsfx.control.MaskerPane

class MainWindowCtrl: GvBaseWindowCtrl() {
    @FXML private lateinit var mainSplit: SplitPane
    @FXML private lateinit var menuBar: AnchorPane
    @FXML private lateinit var branchList: AnchorPane
    @FXML private lateinit var commitList: AnchorPane
    @FXML private lateinit var commitInfo: AnchorPane
    @FXML private lateinit var statusBar: AnchorPane
    @FXML private lateinit var masker: MaskerPane

    fun initialize() {
        mainSplit.setDividerPositions(
            SystemModal.mainSplitPos[0], SystemModal.mainSplitPos[1])
        mainSplit.dividers[0].positionProperty().addListener { _, _, value
            -> SystemModal.mainSplitPos[0] = value.toDouble() }
        mainSplit.dividers[1].positionProperty().addListener { _, _, value
            -> SystemModal.mainSplitPos[1] = value.toDouble() }

        branchList.children.add(BranchList.root)
        commitList.children.add(CommitList.root)
        commitInfo.children.add(CommitInfo.root)
        menuBar.children.add(MenuBar.root)
        statusBar.children.add(StatusBar.root)
    }

    fun runTask(proc: () -> Unit) {
        val scene = MainWindow.root.scene
        scene.cursor = Cursor.WAIT
        val task = object: Task<Unit>() {
            override fun call() {
                try {
                    proc()
                } catch(e: Exception) {
                    Platform.runLater { ErrorDialog(e).showDialog() }
                }
                finally { Platform.runLater { scene.cursor = Cursor.DEFAULT } }
            }
        }
        masker.visibleProperty().bind(task.runningProperty())
        Thread(task).start()
    }
}
{{< /card-code >}}
