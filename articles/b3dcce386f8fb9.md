---
title: "KotlinでKomapperとDomaが共存した環境を作る"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["komapper" , "doma", "kotlin", "gradle", "kapt", "ksp"]
published: true
---

# 前提

| name     | version       |
|:---------|:--------------|
| Doma     | 2.54.0        |
| Komapper | 1.14.0        |
| Kotlin   | 1.9.10        |
| KSP      | 1.9.10-1.0.13 |
| Gradle   | 8.3           |
| JDK      | 20.0.2-tem    |

# Doma(Kapt)とKSPを使った他ライブラリとの共存での問題

KomapperなどKSPを使ったものと共存している場合、`gradle build`をすると、以下エラーメッセージが出ることがあるようです。
`gradle kaptkotlin`をしてから`gradle build`をすると、正常にビルドできはしますが…。

```bash
momose@momose-B650-PG-Lightning-7950x:~/IdeaProjects/experimental-doma$ gradle clean
<<-<
BUILD SUCCESSFUL in 875ms
1 actionable task: 1 up-to-date
le momose@momose-B650-PG-Lightning-7950x:~/IdeaProjects/experimental-doma$ gradle build
> Task :kaptKotlin FAILED

FAILURE: Build failed with an exception.

* What went wrong:
A problem was found with the configuration of task ':kaptKotlin' (type 'KaptWithoutKotlincTask').
  - Gradle detected a problem with the following location: '/home/momose/IdeaProjects/experimental-doma/build/tmp/kapt3/classes/main'.

    Reason: Task ':kspKotlin' uses this output of task ':kaptKotlin' without declaring an explicit or implicit dependency. This can lead to incorrect results being produced, depending on what order the tasks are executed.

    Possible solutions:
      1. Declare task ':kaptKotlin' as an input of ':kspKotlin'.
      2. Declare an explicit dependency on ':kaptKotlin' from ':kspKotlin' using Task#dependsOn.
      3. Declare an explicit dependency on ':kaptKotlin' from ':kspKotlin' using Task#mustRunAfter.

    For more information, please refer to https://docs.gradle.org/8.3/userguide/validation_problems.html#implicit_dependency in the Gradle documentation.

* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.
> Get more help at https://help.gradle.org.

BUILD FAILED in 845ms
5 actionable tasks: 5 executed
```

## 解決方法

kaptとkspの相性問題らしく、build.gradle.ktsのpluginsでの記述順序を変えると解決するようです。
kspと上にkaptを下にすると直るようです。

```kotlin
    id("com.google.devtools.ksp") version "1.9.10-1.0.13"
    kotlin("kapt") version "1.9.10"
```
参考：https://github.com/google/ksp/issues/1445#issuecomment-1763422067

こちらに参考コードを載せています
https://github.com/momosetkn/experimental-doma/blob/650cb12c745a5bf0f8e604c46a6e6240a2f82ec2/build.gradle.kts#L15-L16

# KotlinでDomaのエンティティを書く方法

`immuabtle = true, metamodel = Metamodel()`にするのがポイントなようです。

```kotlin
@Entity(immuabtle = true, metamodel = Metamodel())
@Table(name = "companies")
data class Companies(
    @Id @Column(name = "id") val id: String,
    @Column(name = "name") val name: String,
    @Column(name = "product_id") val productId: String,
)
```

参考記事：
[DOMA2 x Kotlin x SpringBoot でCriteria APIを使う](https://zenn.dev/fujishiro/scraps/ef74544b4120fb)
[Kotlin サポート — Doma 2\.0 ドキュメント](https://doma.readthedocs.io/en/2.19.2/kotlin-support/)

こちらに参考コードを載せています
https://github.com/momosetkn/experimental-doma/blob/4a2118a1c47f139b381058c69361d58b4fd08588/src/main/kotlin/momosetkn/infras/doma/entities/entities.kt#L11-L21

# DomaでKotlin向けのラッパーを使う

build.gradle.ktsのdependenciesに以下を追加する。

```kotlin
implementation("org.seasar.doma:doma-kotlin:2.54.0")
```

Java版（これもKotlinから使うことは可能）
`org.seasar.doma.jdbc.criteria.Entityql`
`org.seasar.doma.jdbc.criteria.NativeSql`

Kotlin版
`org.seasar.doma.kotlin.jdbc.criteria.KEntityql`
`org.seasar.doma.kotlin.jdbc.criteria.KNativeSql`

Kotlin版のorg.seasar.doma.kotlinを使うことにより、JavaのAPIがKotlinでNullable扱いになったりしないようになる。
