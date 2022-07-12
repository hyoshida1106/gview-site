---
tags: ["JavaFX"]
title: "Idle Timer"
date: 2022-01-10
description: >
    Describes the implementation of an idle timer using JavaFX.
---

gview saves the window size in order to restore the previous window size when the application is launched.
Though there are various possible timing for saving, I try to record the size at the timing when no screen operation is performed for a certain period of time (currently 1 second) after the window size is changed.

After searching for a mechanism to achieve that, I found an example that uses JavaFX's Animation Timeline to implement it. So, I used that as a reference to create a generic idle timer, which described here.

---
### IdleTimer Class

My implementation of idle timer class is as follows:

{{< card-code header="**GvIdleTimer.kt**" lang="kotlin" >}}package gview.view.framework
class GvIdleTimer(idleTime: Int, handler: () -> Unit ) {

    private val idleTimeline: Timeline = Timeline(
        KeyFrame(Duration(idleTime.toDouble()), { _ -> handler() }))

    private val userEventHandler = EventHandler<Event> { resetTimer() }

    fun register(scene: Scene, eventType: EventType<Event> = Event.ANY) {
        scene.addEventFilter(eventType, userEventHandler)
        startTimer()
    }

    fun register(node: Node, eventType: EventType<Event> = Event.ANY) {
        node.addEventFilter(eventType, userEventHandler)
        startTimer()
    }

    fun unregister(scene: Scene, eventType: EventType<Event> = Event.ANY) {
        stopTimer()
        scene.removeEventFilter(eventType, userEventHandler)
    }

    fun unregister(node: Node, eventType: EventType<Event> = Event.ANY) {
        stopTimer()
        node.removeEventFilter(eventType, userEventHandler)
    }

    private fun resetTimer() {
        idleTimeline.playFromStart()
    }

    private fun startTimer() {
        idleTimeline.cycleCount = 1;
        idleTimeline.playFromStart()
    }

    private fun stopTimer() {
        idleTimeline.stop( ) 
    }
}
{{< /card-code >}}<br/>

The first step is to declare the class:
```kotlin
class GvIdleTimer(idleTime: Int, handler: () -> Unit )
```

This `GvIdleTimer` class takes two parameters: "*idleTime*" for idle monitoring time(seconds)" and "*handler*" for idle execution handler. 
The sequence is that if no event occurs for the time specified by *idleTime*, *handler* will be called.

To achieve it, create a `javafx.animation.timeline` instance like the following:
```kotlin
    private val idleTimeline: Timeline = Timeline(
        KeyFrame(Duration(idleTime.toDouble()), { _ -> handler() }))
```

The parameters are a `javafx.util.Duration` instance whose parameter is a `Double` conversion of *idleTime*, and a `javafx.event.EventHandler` that calls *handler*.
These make it possible to call the *handler* after *idleTime* seconds.

The next thing we need is a mechanism to restart the timer when an event occurs.
First, we declare the handler instance for receiving events as follows:
```kotlin
    private val userEventHandler = EventHandler<Event> { resetTimer() }
```

This event handler will be registered as an event filter in `javafx.scene.Scene` for idle monitoring.
Once this is registered, the timer can be started to monitor idle.
If an event occurs before the specified idle time is reached, we can simply restart the timer in the event handler.

The method that does this is the `register` method:
```kotlin
    fun register(scene: Scene, eventType: EventType<Event> = Event.ANY) {
        scene.addEventFilter(eventType, userEventHandler)
        idleTimeline.cycleCount = 1;
        idleTimeline.playFromStart()
    }
```

After setting up an event filter for the *Scene* instance to be monitored, start the timer.
Set `cycleCount` to 1, since the timer only needs to run once.

If an event is issued while the timer is being monitored, we call the following handler to restart the timer.
```kotlin
    private fun resetTimer() {
        idleTimeline.playFromStart()
    }
```

---
### Usage

Here is an example for use in a JavaFX application class.

```kotlin

class MyApplication : Application() {

    private val monitor = GvIdleTimer(1000) {
        println("idle event")
    }

    override fun start(stage: Stage) {
        ......
        ......
        monitor.register(stage.scene)
    }
}
```

First, we define an idle timer instance and describe the process to be executed when the time expires.   
The constructor parameter specifies the wait time in ms.
In the example above, 1,000 ms = 1 second is specified.

All left is to register the ``scene`` with the timer instance.
After 1 second of no operation, the defined process will be executed.
