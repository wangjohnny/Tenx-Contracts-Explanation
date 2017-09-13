# Tenx智能合约解读

### Tenx智能合约源码
  智能合约地址：[0xB97048628DB6B661D4C2aA833e95Dbe1A905B280](https://etherscan.io/address/0xB97048628DB6B661D4C2aA833e95Dbe1A905B280)，点击合约地址，
  在打开的页面上，点击名称为Contract Source的Tab页，就可以看到Tenx的智能合约的源码了

### Tenx智能合约类图结构
![image](https://github.com/wangjohnny/Tenx-Contracts-Explanation/raw/master/tenx-pay-model.png)

### 合约解析(按源码顺序解释)
1. Contarct Ownable：定义了智能合约的owner，留给需要用到owner权限控制的合约继承，基本上，正式发行的ICO合约都会需要owner的权限控制，所以这个合约是必须要有的。在tenx合约里，Paytoken与MainSale都继承了这个合约
2. Contarct Authorizable：定义了授权体系，在函数上，添加修饰符(modifier)onlyAuthorized，就可以判断合约里的某个函数是否可以被某人调用，合约的部署人默认就是第一个被授权的人，在合约被部署后，也可以调用addAuthorized，添加新的授权人，这个函数可以被外部合约执行。MainSale继承了这个合约，在其authorizedCreateTokens函数上添加了onlyAuthorized修饰符，这个函数的存在背景是：由于MainSale合约的createTokens函数，对于任何人，只要存入ether，合约就会自动铸造token（当然这个函数还添加了别的约束，比如是否达到上限或者ICO是否已经结束）。但是对于那些手持比特币或者其他ERC20货币的人，没办法自动铸造新货币，怎么办呢？我想大致流程可能是这样的：Tenx某个官方人员(也可能是合约的owner)通道其他渠道收到了某个购买人的比特币(或者ERC20货币)，然后owner通过执行addAuthorized函数，把这个官方人员的账号添加到授权列表，然后这个官方人员就可以为这个购买者铸造新的token，从实际运行代码来看，这个铸造新token的逻辑，没有任何数量上的约束，这样的设置，给Tenx官方保留了最大的方便性，只要他想，就可以随意铸造更多的token。
3. Contarct ExchangeRate：这个合约的功能是设置其他虚拟币(比如ether)与当前token的汇率，它继承了Ownable，而且ExchangeRate的更新方法都添加onlyOwner修饰符，表明只有owner可以更新兑换汇率。MainSale使用了这个合约，并提供了一个setExchangeRate方法，保证了owner可以随时设置token之间的汇率。
4. Library SafeMath：这是一个solidity语言的库，实现了数学的安全型四则运算，针对计算结果可能存在溢出情况，再回头逆向计算一次，查看结果是否相等，来判断是否存在溢出情况。
5. Contarct ERC20Basic：定义了符合ERC20标准的基本函数。
6. Contarct ERC20：定义了符合ERC20标准的全部函数，继承了ERC20Basic，在ERC20Basic的基础上，添加了符合ERC20标准的授权转账函数，owner可以授信一定额度的token给官方内部人员，让他们可以对外转账。
7. Contract BasicToken：实现了ERC20Basic合约中定义的所有函数。
8. Contract StandardToken：继承了ERC20Basic与ERC20，实现了ERC20合约中定义的权转账函数，任何人都可以调用approve方法，授权给某个人代表自己转移token给其他账号。在token的合约体系里，这种授权体系，可以很方便控制token的迁移，但是作用不大。
9. MintableToken：继承了StandardToken与Ownable，ERC20标准只是定义了了一套静止状态的token体系与转移token的功能，但是，没有新Token的铸造机制，MintableToken添加了动态铸造新token的机制，owner可以在ICO发行期内调用mint方法铸造新token，也可以调用finishMinting，结束ICO。
10. PayToken：继承了MintableToken，这个合约是最终将要发行的token，定义了token的名字，symbol（token的缩写词），最小单位等基本信息，以及启动token可以交易的开关。这里需要注意区分MintableToken合约里的mintingFinished变量与PayToken里的tradingStarted变量的功能，前者标识铸造token是否结束(也可以认为是ICO的结束标志)，后者标识token是否交易。token的基本操作流程是先铸造，后交易。
11. MainSale：继承了Ownable与Authorizable，这是tenx众筹的入口，主要实现的功能有：设置了众筹的开始与结束时间，众筹需要获取的ETH数量，调节参数（altDeposits，可以通过调节这个参数，来达到提前完成众筹，或者减少众筹ether的数量）。主要函数功能&变量说明如下：
    ###### createTokens 函数的功能：任何人都可以调用createTokens函数，存入ether，智能合约根据存入的ETH，以及当前token与eth的汇率，来发行新token。
    ###### authorizedCreateTokens 函数的功能：由于比特币没办法在非以太坊平台上转账，假如购买人想用比特币来购买，根据购买者花费的比特币数量，计算好需要发行的token数量，被owner授权的人可以直接调用此函数，达到发行token的目的。setHardCap设置众筹必须要收到的ether数量，
    ###### setStart设置开始时间，假如Tenx官方想要提前开始众筹的话，就不需要等到系统内置的时间了，==这个函数有问题，没有判断时间参数是否在当前时间的前面，由于PAY的合约设置的持续时间是28天，假如owner不小心设置的开始时间是在当前事前的28天之前，那这个行数的执行，会直接让众筹结束。==
    ###### retrieveTokens函数，允许其他人转移符合ERC20标准的token到当前合约地址，然后这个函数再把转进来的token给转回去，我想这个功能应该是防止有人误转其他token到当前智能合约。
    ###### setMultisigVault函数设置多重签名地址。
    ###### retrieveTokens函数是留给owner执行的，把owner名下的token转移到一个多重签名的地址，这个函数可以多次重复执行。
    ##### altDeposits是留给设置非ICO渠道收到的等量货币，用来调节筹集的虚拟货币的价值。

### 疑问点&知识点(感觉了解这些有助于看懂智能合约的代码)：
1. 针对不同的销售渠道，owner以不同的价格售卖token：先执行setExchangeRate方法设置兑换比率，再执行createTokens。setExchangeRate只有owner才可以执行，createTokens任何人都可以执行，理论上这里是存在漏洞的，存在时间差的问题。只要谁知道owner执行了setExchangeRate函数，只要价格合适，那黑客可以利用这个时间执行createTokens给自己铸造更便宜的Token

2. 设置开始时间，是否可以设置多次，假如把时间设置为以前的某天，而且超过28天，是否会导致不可预知的问题。

3. setHardCap，这个函数多次调用，是否会有问题？

4. 暂时没看懂retrieveTokens函数的ERC20(_token)

5. ERC20 带有allowance功能的接口定义，allowance是一种token发放的内部管理授权机制，token的owner可以给管理层授权一定的额度，让管理层可以transfer给别人

