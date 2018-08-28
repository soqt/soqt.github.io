---
layout: post
title: Liquidity对赌智能合约练习
categories: Liquidity
description: 
keywords: liquidity, tezos, smart contract
---


[Liquidity编辑器](http://www.liquidity-lang.org/zeronet/)

开始
----
合约最大的挑战是创造一个密码学安全的随机数，但由于随机数很难在链上创造出来，我们需要要求参与者每人提供一个数再结合起来。两个数结合起来的结果用于判定胜负。
但这个方法显然有很大的缺点，因为判定算法是公开的，所以造成了第二个人能决定游戏的胜负。为了解决这个问题，我们要求游戏参与者根据自己的数字算一个哈希值，并且预先提交他们。所有人哈希值都提交后，再要求他们提交原本的数字，胜负就会被判定。
因为哈希是单向的，所以输入的数字很难被算出，游戏参与者几乎不可能在提交自己数字之前算出对方已经提供的哈希值来影响结果。

数据类型
---------

我们定义游戏参与者的数据类型如下，在Liquidity中，这个数据类型叫`record`, 非常像Go语言中的`struct`, 它包含一些数据字段。
```
type player = {
  k : key;
  h : bytes;
  po : nat option;
}
```

变量包含所有可能变量类型的一种
```
type p =
  | Register of bytes
  | Preimage of nat
  | Resolve of unit
```

合约进入点
```
let%entry main
    (parameter : p)
    (storage : s)
  : operation list * s =
  (* 合约代码在这里 *)
```
这里几个需要注意的点：
1. parameter和storage的数据类型必须明确指明
2. 返回的是一个tuple类型，包括一个内部操作的list和storage的数据类型

参数的类型可以在函数外定义，但是需要在main函数之前被声明。当我们在写liquidity智能合约代码的时候，最好的方法是先不要用function，而是直接把逻辑关系用一整段代码表示出来。因为编译后的函数会占用很大的空间，尤其是当他们有很多参数的时候。


###游戏数据存储

```
type s = {
  one : player option;
  two : player option;
}
```
因为合约的开始并没有参与者，所以要规定option type，play是我们上面自己定义的type。option type可以包括一个定义的数据(这里是`play`)或者`None`类型。我们可以通过`math`来确定当前的数据存储类型。

```
(* match statement *)
begin match storage.one with
| None -> (* do something *)
| Some x -> (* do something else *)
end
```

我们之前定义了一个数据类型`p`,这个类型可以是 Rigister, Preimage, Resolve类型的一种。想一下我们的游戏进程。首先，游戏参与者进入游戏，并用一个哈希值注册进入，这是p的类型为Register; 当他公开自己的数字的时候，这时p的类型变为nat；最后当游戏胜负判定后，p的类型转变为Resolve。因为三种类型不可以同时存在，所以我们可以用一个变量类型代表所有的可能类型。

变量会被编译器编译为Michelson中的 `or`. `record` 和 `tuple` 则是被编译为`pair`

我们可以用Current.sender ()来辨别合约参与者。()这里是代表的unit类型，是一个占位符，用于我们不需要任何数据类型的时候(有些像别的语言中void的作用)。

Current.sender gives us an address, which can be compared to other addresses. If we want to transfer to the contract at that address, we have to let the compiler know what parameter type it should expect with Contract.at. If the address contains a contract with another parameter type, we will get a runtime error.


但两个参与者注册后，他们需要提供自己的数字。数字会被根据已提交的哈希值来确认是否正确并存入storage。我们可以用之前保存的地址来确定只有合约参与人可以查看他们提供的数字。


写一下吧
----------
合约执行的时候，需要按照match来执行合约，如果match不到则会报错。如果操作正确，我们会保存玩家地址并且更新对应的storage数据。

storage可以被直接更改，或者可以新建一个storage然后跟着internal operations返回出去。

我们用`xor`来合并两个数字。因为xor对于bytes无效，所以我们使用数字来判定胜负。我们把结果除以2然后查余。因为分母不能为0，当分母为0的时候，返回的operation type为`None`。当除法合规时，则会返回`Some (result, remainder)`。根据余数我们来分辨哪个游戏参与者获胜。完整代码在下方，可以用[Liquidity编辑器](http://www.liquidity-lang.org/zeronet/)编译。可以使用test或debug深入了解代码。


部署智能合约到区块链
--------------------
部署Michelson合约时，我们需要init storage。
```
let%init storage = {
  one = (None : player option);
  two = (None : player option);
}
```
将合约编译为Michelson，你可以直接在Liquidity上面部署到区块链上面。
spendable和delegatable两个选项决定这个合约里的钱是否能被合约创建人花费。如果设置了spendable，那么合约创建人则可以花费智能合约里的tezzies。如果你在写一个面向大众的智能合约的时候，你可能不希望将它设置为spendable，因为合约因为里面的钱可以随意花费而失去了公信力。delegatable为是否可以将合约里的钱委托给烘焙。

当合约部署后。你可以在`Examine`里面查看合约的storage，并可以在`Call`里面呼叫合约.





```
type p =
  | Register of bytes
  | Preimage of nat
  | Resolve of unit
```


此练习参考Martin Pospech的[文章](https://martin.pospech.cz/post/getting_started_with_liquidity/)