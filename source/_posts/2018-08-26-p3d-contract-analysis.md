---
layout: post
title: P3D合约代码分析
categories: [ethereum, Smart Contract, solidity]
description: p3d smart contract code breakdown
keywords: p3d, fomo3d, solidity
date: 2018-08-26 00:00:00
hidden: true
---

前一阵大热的Fomo3D开创了智能合约庞氏骗局的先河，开创性的玩法让大量资金疯狂涌入这个游戏。F3D的头奖现已开出，结局是一名黑客利用区块内Gas限量的特性堵塞区块从而不让其他交易写入区块，最后成功拿走千万大奖。黑客技术分析改天再谈, 今天我们来深入分析一下P3D这个用户股份合约的内容。

<!-- more -->

合约结构
-------

```
contract Hourglass {
    modifier onlyBagholders()
    modifier onlyStronghands()
    modifier onlyAdministrator()

    modifier antiEarlyWhale(uint256 _amountOfEthereum)

    /*=====================================
    =            CONFIGURABLES            =
    =====================================*/
    string public name = "PowH3D";
    string public symbol = "P3D";
    uint8 constant public decimals = 18;
    uint8 constant internal dividendFee_ = 10;
    uint256 constant internal tokenPriceInitial_ = 0.0000001 ether;
    uint256 constant internal tokenPriceIncremental_ = 0.00000001 ether;
    uint256 constant internal magnitude = 2**64;
    
    // proof of stake (defaults at 100 tokens)
    uint256 public stakingRequirement = 100e18;
    
    // ambassador program
    mapping(address => bool) internal ambassadors_;
    uint256 constant internal ambassadorMaxPurchase_ = 1 ether;
    uint256 constant internal ambassadorQuota_ = 20 ether;
    

    /*================================
    =            DATASETS            =
    ================================*/
    // amount of shares for each address (scaled number)
    mapping(address => uint256) internal tokenBalanceLedger_;
    mapping(address => uint256) internal referralBalance_;
    mapping(address => int256) internal payoutsTo_;
    mapping(address => uint256) internal ambassadorAccumulatedQuota_;
    uint256 internal tokenSupply_ = 0;
    uint256 internal profitPerShare_;
    
    // administrator list (see above on what they can do)
    mapping(bytes32 => bool) public administrators;
    
    // when this is set to true, only ambassadors can purchase tokens (this prevents a whale premine, it ensures a fairly distributed upper pyramid)
    bool public onlyAmbassadors = true;
    
    /*=======================================
    =            PUBLIC FUNCTIONS            =
    =======================================*/
    // 合约进入点
    function Hourglass() public

    function buy(address _referredBy) public payable returns(uint256)
    // fallback
    function() payable public
    function reinvest() onlyStronghands() public
    function exit() public
    function withdraw() onlyStronghands() public

    function sell(uint256 _amountOfTokens) onlyBagholders() public
    function transfer(address _toAddress, uint256 _amountOfTokens) onlyBagholders() public returns(bool)


    function totalEthereumBalance() public view
    function totalSupply() public view returns(uint256)
    function myTokens() public view returns(uint256)

    function myDividends(bool _includeReferralBonus) public view returns(uint256)
    function balanceOf(address _customerAddress) view public returns(uint256)
    function dividendsOf(address _customerAddress) view public returns(uint256)
    function sellPrice() public view returns(uint256)
    function buyPrice() public view returns(uint256)
    function calculateTokensReceived(uint256 _ethereumToSpend) public view returns(uint256)
    function calculateEthereumReceived(uint256 _tokensToSell) public view returns(uint256)

    /*==========================================
    =            INTERNAL FUNCTIONS            =
    ==========================================*/
    function purchaseTokens(uint256 _incomingEthereum, address _referredBy) antiEarlyWhale(_incomingEthereum) internal returns(uint256)
    function ethereumToTokens_(uint256 _ethereum) internal view returns(uint256)
    function tokensToEthereum_(uint256 _tokens) internal view returns(uint256)
}
```
通过函数名称可以大概看出每个函数负责的操作，我们先走一遍流程。
假设你要购买1 eth价值的P3D代币，这时你需要调用buy()函数，并携带邀请人地址`_referredBy`, 如没有调用则进入fall back 函数。h