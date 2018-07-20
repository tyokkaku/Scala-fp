# パーサーコンビネーターで作るJSONパーサー

## 必要なコンビネーターの機能

- 必要なパーサー
  - 文字列パーサー s、 "null"などの値や"["などの記号に使う
  - 文字列リテラルパーサー stringLiteral
  - 浮動小数点数パーサー floatingPointNumber
- 必要なコンビネーター
  - 選択 |
  - 逐次合成 ~
  - 関数の適用 ^^
  - 逐次合成して右側の結果を捨てる <~
  - 逐次合成して左側の結果を捨てる ~>
  - 繰り返し rep
  - セパレータ付き繰り返し repsep

## 暗黙を使ったコンビネーターライブラリ

選択 |

```scala
/**
  * select
  *
  * @param right 選択を行うパーサー
  * @return
  */
def |[U >: T](right: => Parser[U]): Parser[U] = input => {
  parser(input) match {
    case success@Success(_, _) => success
    case Failure => right(input)
  }
}
```

逐次合成 ~

```scala
/**
  * combine
  *
  * @param right 逐次合成を行うパーサー
  * @tparam U パーサーの結果
  * @return
  */
def ~[U](right: => Parser[U]): Parser[(T, U)] = input => {
  parser(input) match {
    case Success(value1, next1) =>
      right(next1) match {
        case Success(value2, next2) =>
          Success((value1, value2), next2)
        case Failure =>
          Failure
      }
    case Failure =>
      Failure
  }
}
```

関数の適用 ^^

```scala
/**
  * map
  *
  * @param function 適用する関数
  * @tparam U パーサーの結果の型
  * @return
  */
def ^^[U](function: T => U): Parser[U] = input => {
  parser(input) match {
    case Success(value, next) => Success(function(value), next)
    case Failure => Failure
  }
}
```

## 左の結果のみを合成、右の結果のみを合成

左と右の取捨

```scala
/**
  * use left
  *
  * @param right 右側のパーサー
  * @return パーサーの結果の型
  */
def <~(right: => Parser[Any]): Parser[T] = input => {
  parser(input) match {
    case Success(value1, next1) =>
      right(next1) match {
        case Success(value2, next2) =>
          Success(value1, next2)
        case Failure =>
          Failure
      }
    case Failure =>
      Failure
  }
}

/**
  * use right
  *
  * @param right 右側のパーサー
  * @tparam U パーサーの結果の型
  * @return
  */
def ~>[U](right: => Parser[U]): Parser[U] = input => {
  parser(input) match {
    case Success(value1, next1) =>
      right(next1) match {
        case Success(value2, next2) =>
          Success(value2, next2)
        case Failure =>
          Failure
      }
    case Failure =>
      Failure
  }
}
```

## セパレーター付き繰り返し合成

セパレータ付き繰り返し合成

```scala
  def rep1sep[T](parser: => Parser[T], sep: Parser[String]): Parser[List[T]] =
    parser ~ rep(sep ~> parser) ^^ { t => t._1 :: t._2 }

  def success[T](value: T): Parser[T] = input => Success(value, input)

  def repsep[T](parser: => Parser[T], sep: Parser[String]): Parser[List[T]] =
    rep1sep(parser, sep) | success(List())
```

## 浮動小数点リテラルと文字列リテラル

正規表現による簡易的な実装

```scala
  val floatingPointNumber: Parser[String] = input => {
    val r = """^(-?\d+(\.\d*)?|\d*\.\d+)([eE][+-]?\d+)?[fFdD]?""".r
    val matchIterator = r.findAllIn(input).matchData
    if (matchIterator.hasNext) {
      val next = matchIterator.next()
      val all = next.group(0)
      val target = next.group(0)
      Success(target, input.substring(all.length))
    } else {
      Failure
    }
  }

  val stringLiteral: Parser[String] = input => {
    val r = ("^\"(" +"""([^"\p{Cntrl}\\]|\\[\\'"bfnrt]|\\u[a-fA-F0-9]{4})*+""" + ")\"").r
    val matchIterator = r.findAllIn(input).matchData
    if(matchIterator.hasNext) {
      val next = matchIterator.next()
      val all = next.group(0)
      val target = next.group(1)
      Success(target, input.substring(all.length))
    } else {
      Failure
    }
  }
```

## JSON パーサーを定義する

BNFでJSONパーサーを定義する

```scala
// JSON exmaple

{
  "title" :"Hello, JSON!",
  "numbers" : [1,2,3,4],
  "status" : {
    "hoge" : "fuga",
    "piyo" : null
  }
}


// BNF
<value> ::= <obj> | <arr> | <stringLiteral> | <floatingPointNumber> | "null" | "true" | false
<obj> ::= "{" [<members>] "}"
<arr> ::= "[" [<values>] "]"
<members> ::= <member> { "," <member> }
<member> ::= <stringLiteral> ":" <value>
<values> ::= <value> { "," <value> }
```

JSONParser

```scala
object JSONParser extends Combinator {

  def obj: Parser[Map[String, Any]] =
    s("{") ~> repsep(member, s(",")) <~ s("}") ^^ {
      Map() ++ _
    }

  def arr: Parser[List[Any]] =
    s("[") ~> repsep(value, s(",")) <~ s("]")

  def member: Parser[(String, Any)] =
    stringLiteral ~ s(":") ~ value ^^ { t => (t._1._1, t._2) }

  def value: Parser[Any] =
    obj |
      arr |
      stringLiteral |
      (floatingPointNumber ^^ {
        _.toDouble
      }) |
      s("null") ^^ { _ => null } |
      s("true") ^^ { _ => true } |
      s("false") ^^ { _ => false }

  def apply(input: String): Any = value(input)
}
```

BNF記法のルール

```
例
<expr> ::= <term>|<expr><addop><term>

ルール： <expr> が <term> または <expr><addop><term> という形式で書かれる

""    ：文字列は "" で囲む
{}    ：0回以上の繰り返し
*     ：*を後ろにつけたときも、0回以上の繰り返し
+     ：1回以上の繰り返し
[]    ：[]で囲んだ部分は省略可能
()    ：()で囲んだ部分をグループ化する
```

## 課題

### 初級

ArithParser

```scala

```