---
title: "Ktorのauto-reload時にpluginエラーがあると応答しなくなり、原因究明ができない問題のワークアラウンド"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ktor", "kotlin", "auto-reload", "jetbrains", "bug-report", "workaround"]
published: true
---

# 前提とするバージョン

io.ktor:ktor-server-core-jvm:2.3.8
io.ktor:ktor-server-netty-jvm:2.3.8

# 問題

development = trueによる[Auto\-reload](https://ktor.io/docs/auto-reload.html)中に、プラグインにエラーが発生すると、ログが何もでず、Ktorが応答しなくなる。
issueは、以下で起票済みです。
[Ktor becoming unresponsive when a plugin errors during auto\-reload\. : KTOR\-6799](https://youtrack.jetbrains.com/issue/KTOR-6799/Ktor-becoming-unresponsive-when-a-plugin-errors-during-auto-reload.)

# auto-reload時にエラーとなるプラグインに気づく方法

auto-reloadのときは、embeddedServerの戻り値が[ApplicationEngineEnvironmentReloading](https://github.com/ktorio/ktor/blob/2.3.8/ktor-server/ktor-server-host-common/jvm/src/io/ktor/server/engine/ApplicationEngineEnvironmentReloading.kt#L32)というクラスになる。
このクラスに[io.ktor.server.engine.ApplicationEngineEnvironmentReloading#reload](https://github.com/ktorio/ktor/blob/2.3.8/ktor-server/ktor-server-host-common/jvm/src/io/ktor/server/engine/ApplicationEngineEnvironmentReloading.kt#L95-L102)が生えているので、サーバー起動後にそれを実行することでエラーになってくれるので、問題のあるプラグインに気づくことができる。

## では、なぜ実際のauto-reloadではなんでエラーのログが出ないのか？

と思うことでしょう。
前提の説明として、実際のauto-reload時には、reloadメソッドとは別の[io.ktor.server.engine.ApplicationEngineEnvironmentReloading#currentApplication](https://github.com/ktorio/ktor/blob/2.3.8/ktor-server/ktor-server-host-common/jvm/src/io/ktor/server/engine/ApplicationEngineEnvironmentReloading.kt#L104-L141)というメソッドで処理がされていて、ここで必要であればreloadをするという処理が行われているらしい（これ自体は問題ない）。
で、currentApplicationを呼び出す側のエラーハンドリングがしっかりとされていないまま、Nettyを終了しているのが大まかな原因のようです。

ktorがauto-reload時にpluginでエラーになると、Nettyが`io.netty.channel.AbstractChannelHandlerContext#invokeChannelRead(java.lang.Object)`から例外がスローされ、`io.netty.channel.AbstractChannelHandlerContext#invokeExceptionCaught(java.lang.Throwable)`で、エラーをキャッチしているようです。キャッチした後は、[io.ktor.server.netty.http1.NettyHttp1Handler#exceptionCaught](https://github.com/ktorio/ktor/blob/2.3.8/ktor-server/ktor-server-netty/jvm/src/io/ktor/server/netty/http1/NettyHttp1Handler.kt#L90)へ戻ってきて、エラー処理は何もせずに、[io.ktor.server.netty.http1.NettyHttp1Handler#exceptionCaughtで、io.netty.channel.ChannelHandlerContext#close()を呼んで](https://github.com/ktorio/ktor/blob/2.3.8/ktor-server/ktor-server-netty/jvm/src/io/ktor/server/netty/http1/NettyHttp1Handler.kt#L104)Nettyを終了しているというのが原因です。

# ワークアラウンド

`start(await = true)`での例です。

```kotlin
fun main() {
    val server = embeddedServer(
        Netty,
        commandLineEnvironment(arrayOf("-config=application.conf"))
    )

    // 自動リロードできないプラグインのエラーに気づけます
    ensureReloadableAsync(server.environment)

    server.start(wait = true)
}

private fun ensureReloadableAsync(environment: ApplicationEngineEnvironment) {
    Thread {
        Thread.sleep(1_000)
        if (environment is ApplicationEngineEnvironmentReloading) {
            environment.reload()
        }
    }.start()
}
```

`start(await = false)`、（起動後、blockしない）場合の実装例は以下を参考にしてください。

https://github.com/momosetkn/ktor-auto-reload-error-logging/blob/b4c1c852defe29bf1806c28c75db260f6dc15da6/src/main/kotlin/com/example/WorkaroundApplication.kt
