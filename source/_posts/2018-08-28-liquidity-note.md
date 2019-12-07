---
layout: post
title: liquidity learning note
categories: [liquidity, smart contract]
description: my note while learning liquidity
keywords: tezos, smart contract, liquidity 
date: 2018-08-28 00:00:00
hidden: true
---

*上次更新于2018-08-28*

<!-- more -->

Doc
----
[Michelson](https://tezos.gitlab.io/zeronet/whitedoc/michelson.html)
[Liquidity](http://www.liquidity-lang.org/doc/)


语法
---------

智能合约接收两个参数，分别为`parameter`和`storage`, 每一个参数必须与智能合约接入点`main`中`parameter`和`storage`的type相符。
例如下面的智能合约，`parameter`的type为`string`,`storage`的type为`map`,语法为`Map [("", 0)]`

liquidity demo code:
```
[%%version 0.3]

let%init storage (myname : string) =
  Map.add myname 0 (Map ["ocaml", 0; "pro", 0])

let%entry main
    (parameter : string)
    (storage : (string, int) map) =

  let amount = Current.amount() in

  if amount < 5.00tz then
    Current.failwith "Not enough money, at least 5tz to vote"
  else
    match Map.find parameter storage with
    | None -> Current.failwith "Bad vote"
    | Some x ->
        let storage = Map.add parameter (x+1) storage in
        ( ([] : operation list), storage )
```
上面这个智能合约是一个投票相关的只能合约，用户可以投票给storage中现有的队伍。
`parameter`是队伍的名字，`storage`是初始智能合约所具有的队伍列表。
当合约从`main`进入后，会用`Current.amount()`提取到当前用户呼叫此只能合约所携带的Tezos金额，当小于5个tezos时会判定失败跳错，反之进入else判定。

`match Map.find parameter storage with` 这是一个liquidity的match语句，结合生成的Michelson，
```
DUUP @storage ;
DUUP @parameter ;
GET ;
```
我们发现`Map.find parameter storage`是在storage里面查找是否有parameter对应的这一项，在Michelson中是将storage和parameter分别复制在stack上然后用GET判定。如果找不到结果，则进入`None`状态，这个合约所执行的代码是挑错，并返回“Bad Vote”。可见None更像是try-catch语法中的catch，用于处理用户错误状态。
如果判定成功，我们进入下一步。
`let storage = Map.add parameter (x+1) storage in` 是将Map中拿到parameter的值并加1，然后赋值给storage。
对应的Michelson代码:
```
{
DUUUP @storage ;
PUSH
    int
    1 ;
DUUUP @x ;
ADD ;
DUUUUP @parameter ;
DIP
    {
    SOME ;
    }
    ;
DIIIP
    {
    DROP ;
    }
    ;
UPDATE @storage ;
NIL
    operation
    ;
PAIR ;
}
```
首先复制整个storage入栈, 并把即将要变化的值push进栈(这里对应(x+1)中的1)。你可以试着更改(x+n)中n的值并编译成Michelson。发现这个值确实对应的是`PUSH int 1`这段代码。此时将@x复制入栈并相加。这里的@x对应的是 `Some x`的值，名称随意，x是用之前match语句取回的storage中对应parameter的值。然后将目前的值与1相加并赋值给storage。
用一段话理解这段代码`let storage = Map.add parameter (x+1) storage in`：`拿到当前的storage并将parameter的键值更改并赋值到storage中`

合约最后返回`( ([] : operation list), storage )`, 返回值为`([], [Map [("dsds", 2)]])`。暂不明确最后一句返回的写法为什么这样写。



*未完待续...*