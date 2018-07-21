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

S式のパーサー

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

構文をパースした後に作られる構文木を表すための型

```scala
package jp.ed.nnn.parsercombinator

import scala.util.Random
import scala.util.matching.Regex


case class ChatBot(commands: List[Command])

sealed trait Command {
  def exec(input: String): Boolean
}

case class ReplyCommand(regex: Regex, replays: List[String]) extends Command {
  // 正規表現にマッチしていれば、ランダムに選んだ一つの返答を返す
  override def exec(input: String): Boolean = {
    regex.findFirstIn(input: String) match {
      case Some(_) =>
        println(Random.shuffle(replays).head)
        true
      case None => false
    }
  }
}
```

## パーサーの実装

チャットBotのパーサー

```scala
package jp.ed.nnn.parsercombinator

import scala.util.parsing.combinator._

object ChatBotTextParser extends JavaTokenParsers {

  def chatBot: Parser[ChatBot] =  "(" ~ "chatbot" ~ commandList ~ ")" ^^ {t => ChatBot(t._1._2)}

  def commandList: Parser[List[Command]] = rep(command)

  def command: Parser[Command] = replyCommand

  def replyCommand: Parser[ReplyCommand] =
    "(" ~ "reply" ~ string ~ replyList ~ ")" ^^ {t => ReplyCommand(t._1._1._2.r, t._1._2) }

  def replyList: Parser[List[String]] = "(" ~ rep(string) ~ ")" ^^ { t => t._1._2}

  def string: Parser[String] =  stringLiteral ^^ { s => s.substring(1, s.length - 1) }

  def apply(input: String): ParseResult[ChatBot] = parseAll(chatBot, input)

}
```

## ボットのインタフェースと読み込みの実装

ボットのメインプログラム

```scala
package jp.ed.nnn.parsercombinator

import scala.annotation.tailrec
import scala.io.Source

object ChatBotMain extends App {

  val text = Source.fromFile("./chatbot.txt").mkString
  val chatBot: ChatBot = ChatBotTextParser(text) match {
    case ChatBotTextParser.Success(result, _) => result
    case failure: ChatBotTextParser.NoSuccess => scala.sys.error(failure.toString)
  }
  println("chatBot: " + chatBot)
  println("ChatBot booted.")

  @tailrec
  def checkInput(): Unit = {
    val input = scala.io.StdIn.readLine(">> ")
    if(input.startsWith("exit")) System.exit(0)

    @tailrec
    def execute(input: String, commands: List[Command]): Unit = {
      if (commands.nonEmpty &&  !commands.head.exec(input)) {
        execute(input, commands.tail)
      }
    }
    execute(input, chatBot.commands)
    checkInput()
  }
  checkInput()
}
```

標準入力

```scala
(chatbot
    (reply ".*(うまくいった).*" (
        "うまくいくなんてすごい"
        "やったね"
    ))
    (reply ".*(つかれた).*" (
        "ゆっくり休んでね"
        "大変だったね"
    ))
    (reply ".*" (
        "こんにちは"
        "こちらこそ、こんにちは"
        "よろしくね"
    ))
)
```

## 言語の拡張

現在時刻をパースし、リプライする

```scala
case class TimeCommand(regex: Regex, start: Int, end: Int, zone: String, replays: List[String]) extends Command {
  override def exec(input: String): Boolean = {
    val now = LocalDateTime.now().atZone {
      ZoneId.of(zone)
    }
    val isInTime = start <= now.getHour && now.getHour <= end
    regex.findFirstIn(input) match {
      case Some(_) if isInTime =>
        println(s"${Random.shuffle(replays).head} 現在時刻：${now}")
        true
      case _ => false
    }
  }
}
```

現在時刻のパース

```scala
  def timeCommand: Parser[TimeCommand] = "(" ~ "time" ~ string ~ digits ~ digits ~ string ~ replyList ~ ")" ^^ {
    t => TimeCommand(
      t._1._1._1._1._1._2.r,
      t._1._1._1._1._2.toInt,
      t._1._1._1._2.toInt,
      t._1._1._2,
      t._1._2)
  }

  def digits : Parser[String] = "[0-9]+".r
```

## 外部 DSL と 内部 DSL