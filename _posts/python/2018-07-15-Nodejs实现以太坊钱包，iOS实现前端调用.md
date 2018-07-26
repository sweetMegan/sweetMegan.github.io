---
layout: post
published: true
title: Nodejs实现以太坊钱包
description: "区块链"
---  

## 一、创建项目

> mkdir myWallet 
> cd myWallet
> npm init


将下面的依赖添加到生成的`package.json`文件中

```
 "dependencies": {
    "bignumber.js": "^7.2.1",
    "ejs": "^2.6.1",
    "koa": "^2.5.2",
    "koa-body": "^4.0.4",
    "koa-bodyparser": "^4.2.1",
    "koa-router": "^7.4.0",
    "koa-static": "^5.0.0",
    "koa-views": "^6.1.4",
    "web3": "^1.0.0-beta.34"
  }
  
```

####安装依赖

> npm install

## 二、初始化项目

 1. 新建文件`index.js`,项目入口
 2. 新建目录`controller`,封装请求处理
   * `account.js` 与账户相关的处理
   * `createAccount.js`创建账户处理
   * `transaction.js` 与交易相关的处理
*    新建目录`model`,模型封装
    * `responsedata.js`,封装应答响应对象
*    新建目录`routers`,路由处理
 * `router.js`
* 新建目录`utils`，工具
 * `myUtils.js`
 
## 三、响应对象

`responsedata.js`

```
function response(code, msg, data) {
    this.code = code;//状态码
    this.msg = msg;//状态描述
    this.data = data;//返回的数据
}
module.exports = response;
```

## 四、开启服务

`index.js`

```
const Koa = require("koa");
const app = new Koa();
const koaBody = require("koa-body");
const router = require("./routers/router");
app.use(async (ctx, next) => {
    console.log(`Process ${ctx.request.method} ${ctx.request.url} ...`);
    await next();
});
//解析post请求
app.use(koaBody({
    multipart:true,
}));
app.use(router.routes());
app.listen("3000");
console.log("服务已开启,监听端口:3000");

```     

## 五、设置路由

`router.js`

```
const router = require("koa-router")();
const createAccountController = require("../controller/createAccount");
const accountController = require("../controller/account");
const transactionController = require("../controller/transaction");
//创建账号
router.post("/createaccount",createAccountController.createAccount);
//私钥解锁账户
router.post("/unlockaccountwithprivatekey",accountController.unlockAccountWithPrivateKey);
//keystore解锁账户
router.post("/unlockaccountwithkeystore",accountController.unlockAccountWithKeyStore);
//发起交易
router.post("/createtransaction",transactionController.createTransaction);
//确认交易
router.post("/sendtransaction",transactionController.sendTransaction);
//交易详情
router.post("/transactionstatus",transactionController.transactionStatus);
module.exports = router;

```

项目中所有请求都采用`post`的方式

## 六、实例化`web3`

`myUtils.js`

```
const Web3 = require("web3");
var fs = require("fs");
getWeb3 = ()=>{
    //初始化web3访问节点为私有链节点
    const web3 = new Web3(Web3.givenProvider||"http://localhost:8545");
    //以太坊测试链
    // const web3 = new Web3(Web3.givenProvider || "https://kovan.infura.io/v3/4abf7f8865064ed6b99ca3fdac820921");
    return web3;
};
module.exports={
    getWeb3,
};
```

调用`getWeb3()`获取`web3`对象

>     var web3 = new Web3(Web3.givenProvider||"http://localhost:8545");

* 第一次调用`getWeb3()`方法时`Web3.givenProvider`为空,所以需要设置一个种子节点。
* `http://localhost:8545`为本地私有链地址，[怎么搭建私链](https://sweetmegan.github.io/2018/07/15/%E4%BB%A5%E5%A4%AA%E5%9D%8A%E7%A7%81%E7%BD%91%E5%BB%BA%E7%AB%8B-%E9%80%9A%E8%BF%87%E5%88%9B%E4%B8%96%E5%8C%BA%E5%9D%97%E6%9D%A5%E5%88%9D%E5%A7%8B%E5%8C%96%E5%8C%BA%E5%9D%97%E9%93%BE/)
* `https://kovan.infura.io/v3/4abf7f8865064ed6b99ca3fdac820921`是以太网`kovan`测试链地址

#### 获取测试链地址

打开 `INFURA` https://infura.io/dashboard
注册，登录
登录成功后来到下面页面

![](http://otwxtrtn9.bkt.clouddn.com/%E8%8E%B7%E5%8F%96%E4%BB%A5%E5%A4%AA%E7%BD%91%E6%B5%8B%E8%AF%95%E9%93%BE%E5%9C%B0%E5%9D%80.png)

可以看到主网和其他三个测试链，按需选择`ENDPOINT`下的地址就是我们所需要的地址



## 七、请求处理

####<font color=red>在执行下面操作前需开启一个私链或者测试链</font>

#### 1、创建账户

`createAccount.js`

```
const web3 = require("../utils/myUtils").getWeb3();
const path = require("path");
const fs = require("fs");
var respones = require("../model/responsedata");
//创建钱包
createAccount = async ctx => {
    var responseData = new respones(0,"success",{});
    // console.log(ctx.request.body);
    var body = ctx.request.body;
    //创建钱包
    var account = web3.eth.accounts.create();
    //生成keyStore文件
    //keyStore是将私钥与用户密码拼接,将拼接结果对称加密得到
    var keyStoreJson = account.encrypt(body.pwd);
    //保存keyStore
    //写入文件的keyStore数据
    // var keyStoreStr = JSON.stringify(keyStoreJson);
    // //keyStore文件名
    // var keyStoreFileName = "UTC--"+new Date().toISOString()+"--"+account.address;
    // //文件保存路径
    // var keyStoreFilePath = path.join(__dirname,"../static/keystore",keyStoreFileName);
    // await fs.writeFile(keyStoreFilePath,keyStoreStr,()=>{});
    // var responseData = new respones(0,"success",{
    //     "downloadUrl":"/keystore/"+keyStoreFileName,
    //     "privateKey":account.privateKey
    // });
    responseData.data.keyStore = keyStoreJson;
    ctx.body = responseData;
};
module.exports={
    createAccount
};

```

`keystore`文件在客户端自己保存，服务端不保存


#### 2、账号操作

`account.js`

```

const web3 = require("../utils/myUtils").getWeb3();
const myUtil = require("../utils/myUtils");
var response = require("../model/responsedata");
setResponseDataWithAccount = async(account, responseData)=> {
    responseData.data.address = account.address;
    responseData.data.privateKey = account.privateKey;
    responseData.data.balance = await getBalanceWithAddress(account.address);
    return responseData;
};

getBalanceWithAddress = async(address)=> {
    console.log("account:" + address);
    var balance = await web3.eth.getBalance(address);
    return web3.utils.fromWei(balance, "ether");
};
//使用私钥解锁账户
unlockAccountWithPrivateKey = async(ctx)=> {
    var responseData = new response(0, "success", {});
    var body = ctx.request.body;
    var privateKey = body.privateKey;
    console.log("privateKey:" + privateKey);
    var account = web3.eth.accounts.privateKeyToAccount(privateKey);
    // ctx.body = {name:"解锁"};
    var data = await setResponseDataWithAccount(account, responseData);
    console.log(data);
    ctx.body = data;
};
//使用keyStore文件
unlockAccountWithKeyStore = async(ctx)=> {
    var responseData = new response(0, "success", {});
    var body = ctx.request.body;
    var pwd = body.pwd;
    var keyStore = body.keyStore;
    try {
        var account = web3.eth.accounts.decrypt(keyStore, pwd);
        ctx.body = await setResponseDataWithAccount(account, responseData);
    }catch (error){
        console.log(error)
        responseData.code = 1;
        responseData.message = "failed";
        responseData.data = {error: error.message};
        ctx.body = responseData;
    }
};
module.exports = {
    unlockAccountWithPrivateKey,
    unlockAccountWithKeyStore,
};

```

#### 3、交易处理

`transaction.js`

```
const web3 = require("../utils/myUtils").getWeb3();
var response = require("../model/responsedata");
var bignumber = require("bignumber.js")

//发起交易
createTransaction = async(ctx)=> {
    console.log("createTransaction");
    var responseData = new response(0, "success", {});
    var body = ctx.request.body;
    var fromAddress = body.from;
    var toAddress = body.to;
    //将输入的金额换算成Wei
    var money = web3.utils.toWei(body.money, "ether");
    var gasPrice = await web3.eth.getGasPrice();
    //获取交易的nonce值,一个顺序累加的值
    var nonce = await web3.eth.getTransactionCount(fromAddress);
    var transactionData = {
        from: fromAddress,
        to: toAddress,
        value: money,
        gasPrice: gasPrice,
        data: '0x00',//当使用代币转账或者合约调用时
        nonce: nonce,
    };
    //estimateGas()方法会将transactionData数据做一些操作,导致,transactionData一些值的类型变化,所以下面对transactionData重新赋值
    var gas = await web3.eth.estimateGas(transactionData);
    transactionData = {
        from: fromAddress,
        to: toAddress,
        value: money,
        gasPrice: gasPrice,
        data: '0x00',//当使用代币转账或者合约调用时
        nonce: nonce,
        gas:gas,
    };
    console.log(transactionData);
    responseData.data = transactionData;
    ctx.body = responseData;

};

```
创建一笔交易，返回交易的`gas`和`gasPrice`供用户确认

```
//确认交易
/*
* 1、签名交易
* 2、发送交易
* */
sendTransaction = async(ctx)=> {

    console.log("sendTransaction");

    var responseData = new response(0, "success", {});
    var body = ctx.request.body;
    var transactionData = {
        from: body.from,
        to: body.to,
        value: body.value,
        gasPrice: body.gasPrice,
        data: body.data,//当使用代币转账或者合约调用时
        nonce: body.nonce,
        gas: body.gas
    };
    //privateKey为了安全,需要进行加密处理
    var privateKey = body.privateKey;
    //签名交易
    console.log(transactionData);
    console.log(privateKey);
    var signTransactionData = await web3.eth.accounts.signTransaction(transactionData, privateKey);
    try {
        //发送交易
        await web3.eth.sendSignedTransaction(signTransactionData.rawTransaction, (error, hash)=> {
            if (!error) {
                responseData.data.hash = hash;
            } else {
                responseData.code = 1;
                responseData.message = "failed";
                responseData.data = {error: error.message};
            }
        })
    } catch (error) {
        console.log(error);
        responseData.code = 1;
        responseData.message = "failed";
        responseData.data = {error: error.message};
    }
    ctx.body = responseData
};
```
确认一笔交易，这里才是真正提交一笔交易，这时不会直接返回结果，交易提交后，如果连接的是测试链，需要自己去挖矿,执行命令`miner.start()`,直到矿工确认这笔交易后，才会返回交易信息。如果是测试链，因为它有自己的生态，不需要做其他操作。

```
//根据txHash查询交易状态
transactionStatus = async(ctx)=> {
  var responseData = new response(0,"success",{});
    var data = ctx.request.body;
    var txHash = data.txHash;
    var result = await web3.eth.getTransactionReceipt(txHash);
    if (result != null){
        responseData.data = result;
    }
    ctx.body = responseData;
};
module.exports = {
    createTransaction,
    sendTransaction,
    transactionStatus,
};

```

根据交易hash，返回

> 注 ：`createTransaction`方法中调用`estimateGas()`方法会将`transactionData`数据做一些操作,导致,`transactionData`一些值的类型变化,所以下面对`transactionData`重新赋值







## demo下载 https://github.com/sweetMegan/myWalletDemo

####`myWallet`为nodejs后台
####`myWalletClient`为iOS前端


