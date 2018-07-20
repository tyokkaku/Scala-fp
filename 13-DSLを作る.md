# 13. DSLを作る

## ドメイン特化言語 DSL

DSL(Domain Specific Language)： 特定の用途/ドメインに対してと特化した言語のこと

## S式

S式： Lispで導入されている2分木とリストの構造を表す記述方式のこと

atomic-symbol： 英数字記号で表される文字のこと

```scala
<s-expression> ::= <atomic-symbol> 
      | "(" <s-expression> "." <s-expression> ")" 
      | <list> 
<list>  ::= "(" <s-expression> { <s-expression> } ")"
```

```scala
object SExpressionParser extends JavaTokenParsers {

  def sExpression: Parser[Any] = atomicSymbol | "(" ~ sExpression ~ "." ~ sExpression ~ ")" ^^ (t => (t._1._1._1._2, t._1._2)) | list

  def list: Parser[List[Any]] = "(" ~ sExpression ~ rep(sExpression) ~ ")" ^^ (t => t._1._1._2 :: t._1._2 )

  def atomicSymbol: Parser[String] = "[0-9A-z]+".r

  def apply(input:String):Any = parseAll(sExpression, input)

}
```

## チャットボットの動きを定義する DSL
## 構文構造の実装
## パーサーの実装
## ボットのインタフェースと読み込みの実装
## 言語の拡張
## 外部 DSL と 内部 DSL