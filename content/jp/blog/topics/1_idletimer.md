---
tags: ["JavaFX"]
title: "アイドルタイマ"
date: 2022-01-10
description: >
    JavaFXでアイドルタイマを実装する方法について説明しています。
---

アプリケーションの起動時、前回のウィンドウサイズを復元するために、ウィンドウサイズの保存を行っています。
保存するタイミングはいろいろ考えられますが、このプログラムでウィンドウサイズ変更後、一定時間(現在は1秒)画面操作が行われなかったタイミングでサイズを記録するようにしています。

これを実現する機構をいろいろ探した結果、JavaFXのAnimation Timelineを使って実装している例を見付けたので、それを参考に汎用的なアイドルタイマを作成しました。ここではそれを説明します。

---
### IdleTimerクラス

実装クラスは次のようなものです。

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

まず最初に、クラスを宣言します。
```kotlin
class GvIdleTimer(idleTime: Int, handler: () -> Unit )
```

このクラス`GvIdleTimer`は、“*idleTime*:アイドル監視時間(単位=秒)”、“*handler*:アイドル時実行ハンドラ”という2つのパラメータを取ります。*idleTime*で指定された時間、何もイベントが発生しなければ*handler*が呼び出される、というシーケンスです。

これを実現するために、次のような`javafx.animation.Timeline`インスタンスを作成します。
```kotlin
    private val idleTimeline: Timeline = Timeline(
        KeyFrame(Duration(idleTime.toDouble()), { _ -> handler() }))
```
*idleTime*を`Double`変換した値をパラメータとする`javafx.util.Duration`インスタンスと、*handler*を
コールする`javafx.event.EventHandler`をパラメータとして指定しています。これにより、*idleTime*秒後に*handler*を呼び出すことが可能になります。

次に必要なのは、イベント発生時にタイマを再スタートする仕組みです。まず、イベントを受信するためのハンドラインスタンスを次のように宣言しておきます。
```kotlin
    private val userEventHandler = EventHandler<Event> { resetTimer() }
```

このイベントハンドラは、アイドル監視を行う`javafx.scene.Scene`にイベントフィルタとして登録します。この登録を行った上でタイマを起動すれば、アイドル監視が可能になります。指定されたアイドル時間に到達する前にイベントが発生した場合には、イベントハンドラでタイマを再スタートさせればよいのです。

これを行うメソッドが、次の`register`メソッドです。
```kotlin
    fun register(scene: Scene, eventType: EventType<Event> = Event.ANY) {
        scene.addEventFilter(eventType, userEventHandler)
        idleTimeline.cycleCount = 1;
        idleTimeline.playFromStart()
    }
```
イベント監視を行う*Scene*インスタンスに対してイベントフィルタを設定した上で、タイマを起動します。
タイマは一度だけ実行すればよいので、`cycleCount`には1を設定します。

タイマ監視中にイベントが発行された場合は、以下のハンドラでタイマを再起動します。
```kotlin
    private fun resetTimer() {
        idleTimeline.playFromStart()
    }
```

---
### 使用方法

JavaFXのアプリケーションクラスで使用する場合の例です。

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

アイドルタイマのインスタンスを定義して、処理を記述します。  
コンストラクタには、待機時間をms単位で指定します。
上のリストでは1,000=1秒を指定しています。

後は、タイマインスタンスに``scene``を登録すればOKです。
操作のない状態が1秒継続すると、定義した処理が実行されます。
