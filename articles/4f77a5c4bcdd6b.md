---
title: "Komapperを使い倒してみる"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tech", "Komapper", "kotlin", "ORM", "DB"]
published: true
---

tips集です

# 動的に条件文を作成する処理を共通化する

例として、from-toのPairで、動的にbetweenと<=と>=を切り替えるものを作ってみます。
from-toの両方を指定したり、fromだけ指定したかったり、toだけを指定したかったりということができるようになります。

```kotlin
class ConditionExtension<F : FilterScope<F>>(
    val filterScope: FilterScope<F>
) {
    infix fun <T : Comparable<T>, S : Any> ColumnExpression<T, S>.range(
        fromTo: Pair<T?, T?>,
    ) {
        val column = this
        val (from, to) = fromTo
        return with(filterScope) {
            if (from != null && to == null) {
                column greaterEq from
            } else if (from == null && to != null) {
                column lessEq to
            } else if (from != null && to != null) {
                column between from..to
            }
        }
    }
    // 他にも自前で動的に条件文を作成する共通処理があれば、ここに追加していく
}

fun FilterScope<*>.conditionExtension(
    block: ConditionExtension<*>.() -> Unit,
) {
    ConditionExtension(this).apply {
        block()
    }
}
```

```kotlin
class CompaniesRepository(
    private val db: JdbcDatabase,
) {
    fun findIdAndSameCreatorCountList2(
        fromTo: Pair<LocalDateTime?, LocalDateTime?>,
    ): List<InfraCompanies> {
        val metaCompany = InfraCompanies.meta
        val metaSameCreatorCompany = InfraCompanies.meta
        val mainQuery = QueryDsl.from(metaCompany)
            .leftJoin(metaSameCreatorCompany) {
                metaCompany.createdBy eq metaSameCreatorCompany.createdBy
            }
            .where {
                conditionExtension {
                    metaCompany.createdAt range fromTo
                }
            }
            .groupBy(metaCompany.id)

        val result = db.runQuery {
            mainQuery
        }

        return result
    }
}
```

この実装のコツは、FilterScopeというContextの中に居ないとeqやgreaterEqなどのKomapperの関数が使えないので、whereのブロック中で実行することでFilterScopeを受け取るところです。

https://github.com/momosetkn/experimental-doma/blob/cfce7e4ce9dfa87fbdc89dc6295cc69824ccac9e/src/main/kotlin/momosetkn/infras/komapper/repositories/CompaniesRepository.kt#L189-L272
他にも色々実装しています

# count(distinct hoge)

[v1.17.0](https://github.com/komapper/komapper/releases/tag/v1.17.0)から標準機能でできるようになりました。以下のように書きます。

```kotlin
val mainQuery = QueryDsl.from(metaCompany)
.select(countDistinct(metaCompany.id))
// select count(distinct t0_.id) from companies as t0_
```

実装されていないことを呟いたら、翌日にはリリースされてました…はやい…。

https://twitter.com/nakamura_to/status/1766049794320200069


# count(distinct (hoge1, hoge2))と複数カラムを指定する

これはRDBMS毎の差異が大きいためか、標準機能としてはサポートされていません

Komapperのドキュメントにもある[独自のカラム式](https://www.komapper.org/ja/docs/reference/query/querydsl/expression/#user-defined-expression-column-expression)という機能を使って実装することができます。
columnExpressionのnameは、select句の各列を一意に特定するため項目の一部のようです。
MySQLとPostgreSQLではcount(distinctをちゃんと実装しました。他のRDBMSの事情は知らない（調べてない）ので手抜き実装となっております…。
実装例を示します。

```kotlin
fun countDistinctMultiple(
    vararg expressions: ColumnExpression<*, *>,
): ColumnExpression<Long, Long> {
    val name = "countDistinctMultiple"
    val columns = expressions.map { Operand.Column(it) }
    return columnExpression(Long::class, name, columns) {
        if (dialect.driver == "mysql") {
            append("count(distinct ")
            columns.forEach {
                visit(it)
                append(", ")
            }
            cutBack(2)
            append(")")
        } else if (dialect.driver == "postgresql") {
            append("count(distinct ")
            columns.forEach {
                visit(it)
                append(", ")
            }
            cutBack(2)
            append(")")
        } else {
            append("count(distinct ")
            columns.forEach {
                visit(it)
                append(" || '-' || ")
            }
            cutBack(11)
            append(")")
        }
    }
}
```

```kotlin
val query = QueryDsl.from(metaCompany)
    .leftJoin(metaSameCreatorCompany) {
        metaCompany.createdBy eq metaSameCreatorCompany.createdBy
    }
    .groupBy(metaCompany.id)
    .select(metaCompany.id, countDistinctMultiple(metaCompany.id, metaSameCreatorCompany.id))
```

# クエリの途中でjoinしたentityの一覧を取得する

join済みリストを取得したいときってありますよね。例えばすでにjoinしているのに間違ってもう一度joinしてしまった場合、エラーとなるため、その回避策として必要だと思うことがあります。
このため、すでにjoin済みかどうかを確認して、joinしていない場合のみjoinする処理を書いてみました。
（Komapperの元になったDomaではリフレクションを使ってハックしないとjoinの情報が取れませんでした…）

```kotlin
fun <ENTITY2 : Any, ID2 : Any, META2 : EntityMetamodel<ENTITY2, ID2, META2>> EntitySelectQuery<ENTITY2>.ensureLeftJoin(
    metamodel: META2,
    on: OnDeclaration,
): EntitySelectQuery<ENTITY2> {
    val joins = (this.context as SelectContext<*, *, *>).joins
    return if (joins.any { it.target == metamodel }) {
        this
    } else {
        leftJoin(Relationship(metamodel, on))
    }
}
```

```kotlin
// companyに対して、employeeをjoinされている状態にする
fun SelectQueryBuilder<InfraCompanies, String, _InfraCompanies>.ensureLeftJoinCompany(
    metaSameCreatorCompany: _InfraCompanies,
    metaEmployee: _InfraEmployees,
) = ensureLeftJoin(metaSameCreatorCompany) {
    metaCompany.id eq metaEmployee.companyId
}

val mainQuery = QueryDsl.from(metaCompany)
    .let {
        if (name != null) {
            // 検索条件に応じて、特定の場合のみjoinをする
            it.ensureLeftJoinCompany(metaCompany, metaEmployee).where {
                metaEmployee.name eq name
            }
        } else {
            it
        }
    }
    .let {
        if (email != null) {
            // 検索条件に応じて、特定の場合のみjoinをする
            it.ensureLeftJoinCompany(metaCompany, metaEmployee).where { // ensureLeftJoinでjoinすることで、すでにjoin済みの場合はjoinしない
                metaEmployee.email eq email
            }
        } else {
            it
        }
    }
    .groupBy(metaCompany.id)
    .select(metaCompany.id, countDistinct(metaSameCreatorCompany.id))
```

# テスト結果確認のための再読込機能

標準では存在しません。テスト実行後のentityを取得するために、テスト実行前のentityを元にして、再度取得する処理を書いてみました。更新処理のテストで使いたいですね。
`＠Id`を付与したプロパティを元に再検索をかけて返します。
長いので、GitHubのリンクを貼ります。

再読込の処理
https://github.com/momosetkn/experimental-doma/blob/93927d279df80a4b2255755cb3a35be1c68a2be9/src/test/kotlin/KomapperDatabase.kt

使い方
https://github.com/momosetkn/experimental-doma/blob/3365b6f0e6468545e77f2dc438d99cf9fe171744/src/test/kotlin/KomapperExampleTest.kt#L32
