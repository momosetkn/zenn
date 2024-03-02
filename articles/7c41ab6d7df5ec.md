---
title: "DomaのCriteria APIでFROM句にサブクエリを指定する"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "java", "kotlin", "sql", "doma", "orm" ]
published: true
---

# 前提

以下バージョンから、DomaのCriteria APIでFROM句にサブクエリを指定することができるようになりました。

|                             | version   | リリース日      |
|-----------------------------|-----------|------------|
| org.seasar.doma:doma-kotlin | 2.56.0以降  | 2024-02-20 |
| org.seasar.doma:doma-core   | 2.55.0以降  | 2023-12-23 |

# 使い方

※Kotlinでの書き方を前提にして説明します（Javaで書く場合は、各自変換してください…）。

今回は、以下のSQLを例に説明します。

```sql
select t0_.NAME, t0_.AMOUNT
from (select t2_.DEPARTMENT_NAME AS NAME, sum(t1_.SALARY) AS AMOUNT
      from EMPLOYEE t1_
               inner join DEPARTMENT t2_ on (t1_.DEPARTMENT_ID = t2_.DEPARTMENT_ID)
      group by t2_.DEPARTMENT_NAME) t0_
```

まず、サブクエリとするクエリをNativeSqlで書きます。

```kotlin
val d = Department_()
val e = Employee_()

var subquery: SetOperand<Tuple2<String, Salary>> =
    nativeSql
        .from(e)
        .innerJoin(d) { c -> c.eq(e.departmentId, d.departmentId) }
        .groupBy(d.departmentName)
        .select(d.departmentName, sum(e.salary))
```

サブクエリの結果のEntityを定義します。
さきほど書いたサブクエリの結果と順番・個数が一致するように書きます。

```kotlin
@Entity(immutable = true, metamodel = Metamodel())
data class NameAndAmount(
    @Column(name = "name") val name: String,
    @Column(name = "amount") val amount: Int,
)
```

次に、サブクエリを内包するメインのクエリを書きます。ここで、サブクエリの結果のEntityMetamodelをfrom句のメソッドに指定します。
例では、nativeSqlを使っていますが、entityqlを使っても同じように書けます。

```kotlin
val nameAndAmount = NameAndAmount_()

List<NameAndAmount> list =
    nativeSql
        .from(nameAndAmount, subquery)
        .fetch()
```

これで、サブクエリをfrom句に指定することができます。
group byが設定されているSQLで、groupの数を取得したい場合などでも、以下のように、この方法で取得することができます。
```kotlin
val nameAndAmount = NameAndAmount_()

Int count =
    nativeSql
        .from(nameAndAmount, subquery)
        .select(Expressions.count())
        .single()
```

## unionと組み合わせて使う

以下のように、サブクエリの結果のEntityと同じカラムの順番・個数で、unionのNativeSqlを使って書きます。

```kotlin
var subquery: SetOperand<*> = nativeSql
    .from(d)
    .where { c -> c.eq(d.departmentName, "ACCOUNTING") }
    .select(
        d.departmentName,
        Expressions.literal(1200),
    )
    .union(
        nativeSql
            .from(d)
            .where { c -> c.eq(d.departmentName, "OPERATIONS") }
            .select(
                d.departmentName,
                Expressions.literal(900),
            )
    )
var query: NativeSqlSelectStarting<NameAndAmount> =
    nativeSql.from(t, subquery).orderBy { c -> c.asc(t.name) }
```

# Komapperもよろしく

実は、Domaの後発のORMのKomapperでは、ちょっと前からFROM句にサブクエリを指定することができました。

[\[Kotlin\] KomapperでFROM句にサブクエリを指定する](https://zenn.dev/nakamura_to/articles/25ce603ce54f0c)
