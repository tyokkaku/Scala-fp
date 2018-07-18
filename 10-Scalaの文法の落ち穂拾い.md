# Scalaの文法の落ち穂拾い

## パターンマッチで利用できるクラスを作る unapply

- unapply
  - 実装すれば抽出子として利用できる
  - case classは標準で装備されている

簡易的な unapply の実装。Userは抽出子として使うことができる

```scala
object UnapplyStudy extends App {

  class User(private val name: String, private val age: Int)

  object User {
    def unapply(user: User): Option[(String, Int)] = Some(user.name, user.age)
  }

  def printPatternMatched(obj: AnyRef): Unit = {
    obj match {
      case User(name, age) => println(s"Name: ${name}, Age: ${age}")
      case _ => println("can't extract")
    }
  }

  printPatternMatched(new User("Taro", 17))
}
```

unapplyの拡張。String型とUser型で処理を振り分ける

```scala
object User {

  def unapply(obj: Any): Option[(String, Int)] =
    if (obj.isInstanceOf[User]) {
      val user = obj.asInstanceOf[User]
      Some((user.name, user.age))
    } else if (obj.isInstanceOf[String]) {
      val strs = obj.asInstanceOf[String].split("@")
      val name = strs.headOption
      val age = Try { strs.tail.headOption.map(_.toInt) }.toOption.flatten
      (name, age) match {
        case (Some(n), Some(a)) => Some((n, a))
        case _ => None
      }
    } else {
      None
    }
}

// printPatternMatched(new User("Taro", 17))
// Name: Taro, Age: 17
// printPatternMatched("Jiro@16")
// Name: Jiro, Age: 16
```

## 型エイリアス type

型に別名をつけられる

```scala
// (Int, String)型に、Entryという名前をつける
type Entry = (Int, String)

// Entryをつかった独自の型を作ることもできる
// List[Entry]型に、EntryListという名前をつける
type EntryList = List[Entry]

val list: EntryList = List(1 -> "one", 2 -> "two", 3 -> "three")
list: EntryList = List((1,one),(2,two),(3,three))
```

型パラメータにわかりやすい別名をつける

```scala
class Cell[A](val content: A) {
  type ContentType = A
}
```

## XMLリテラル

## 型情報を扱うリフレクション

## コードの構文木を生成するマクロ