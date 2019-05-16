## 关于混合开发项目的启动的建议
当启动一个混合开发项目时，需要做什么准备？

***

### hybird技术的通识

h5内嵌在webview中必然需要和app原生交互，因此需要约定好交互协议
更具体交互通识可查看这两篇文章：

[app中的webview通识篇上](https://juejin.im/post/5bf35ef26fb9a049bc4c438a)

[浅谈Hybrid技术的设计与实现](http://www.cnblogs.com/yexiaochai/p/4921635.html)


### 需要的配置

1. 统一ua标识，客户端在webview用户标识尾部添加应用标识 ：
``` javascript
    console.log(window.navigator.userAgent)
    // "Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X)   AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1"

    // =>
    // "Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X)   AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1 app-1.4.1"
    //app为应用标识 x.x.x为版本号
```

2. H5跳转app页面依旧使用已设置的 url scheme 跳转:
``` javascript
    // 假如页面需要分享到其他应用，在浏览器中使用urlscheme可以跳转回到本应用
    // 个人主页
    const appPersonPage='app://share.person?userUuid=465431type=1'
```

3. app点击header返回按钮默认行为
```
    1，页面具有history记录时，返回到上一条记录
    2，页面无history记录时，退出当前webview返回到上一级原生页面，有可能是上一个webview或原生页面
```

### 使用交互框架建议：[DSBridge](https://github.com/wendux)

如果是主web前端的框架可以选择react native，weex，cordova+ionic等框架，否则主客户端的项目建议选择这个轻量实用的框架
框架有中文说明，框架提供了交互技术，原生和前端只需要添加和使用对应方法就好了

[DSBridge-IOS](https://github.com/wendux/DSBridge-IOS)

[DSBridge-Android](https://github.com/wendux/DSBridge-Android)  

前端应该持续维护一个bridge.js的项目，项目包含了客户端与js的所有交互方法，提供完善的接口，测试页面和文档。
今后有页面需要接入应用内，则引入这个bridge.js

***

### 方法约定    

##### 方法说明：  
    name:方法名;  
    params:json对象参数；值为：name:Boolean(类型)|required(是否必填)  
    async：是否异步(是：true,否：false),异步方法在完成时将处理结果传入回调函数并调用  
  
##### 回调约定：返回一个json对象，包含以下信息  
    data:返回数据，具体数据定义在data这个json对象中  
    msg:返回的提示信息  
    code:状态码 (错误：0，正常：1，警告：2)  

##### 关于方法对接的安全
    无验证，直接调用有可能存在安全问题，在设计通用接口时，建议使用两端约定的盐值如：abc123456，和其他参数生成签名，双方根据params或者时间戳根据特定算法生成对应签名进行匹配验证，增加接口的安全性

***

### 常用方法功能

#### 1，webview跳转 

**实现功能：**  
```
    1，页面内跳转，与app无关，无需做处理  
    2，h5页面跳转app页面，只限制使用已定义url scheme跳转  
    3，h5跳转h5，打开新的webview  
    在不是同一个h5项目的情况下需要使用第三种情况，所以app需要实现一个方法实现第三种情况  
```

**js调用方法：**  
``` javascript
    /* 
        name:'webviewPush',
        params:{
            url:String|required
        },
        async:false 
    */

    const response = dsBridge.call("webviewPush",{
        url:'www.baidu.com'
    });

    // data:{}  
```

#### 2，获取用户信息  

**实现功能：**  
```
    1，有用户信息则返回  
    2，传入forceLogin为true时，未登陆的用户弹出登录页面要求登录，登录后返回用户信息
    3，无用户信息,或要求登陆但是还是未登陆时（关闭登录界面） ：返回null
```

**js调用方法：**  
``` javascript
    /* 
        name:'getUserInfo',
        params:{
            forceLogin：Boolean
        },
        async:true 
    */

    dsBridge.call("getUserInfo",{
        forceLogin:true
    },function(response){

    });

    // response
    /* 
        data:{
            token:'*',
            userInfo:{}
        }
    */
```

#### 3，获取用户登陆状态  

**实现功能：**  
```
    1，返回用户登陆状态 
    2，已登陆：true,未登陆：false 
```

**js调用方法：**  
``` javascript
    /* 
        name:'isLogin',
        params:{},
        async:false 
    */

    const response = dsBridge.call("isLogin");

    // response
    /* 
        data:{
            isLogin : true || false
        }
    */
```

#### 4，设置header功能键  

**实现功能：**  
```
    在有活动，或者拉新需求的时候需要在header设置一个分享按钮，或者其他原生功能键，点击触发js回调，原生不写业务逻辑
    1，只使用文字按钮，点击后触发回调，按钮排列在右侧
    2，disabled按钮置灰点击无效化（不触发回调）
    3，无按钮情况下创建一个该配置的按钮
    4，有同名按钮情况下使用配置更新该按钮
    5，按钮按index排序，index最大值放置在最右侧
    6，有color参数时使用color参数设置的字体颜色，没有则使用默认header的文字颜色
    7，根据name值移除按钮
    8，关键字文字转换为app内置图标按钮，如使用"分享"时转换为个人主页的分享图标显示(不强求)
```

**js调用方法：**  
``` javascript
    // 设置
    /* 
        name:'setHeaderTextBtn',
        params:{
            disabled:Boolean,
            name:String|required,
            index:Number|required,
            color:String
        },
        async:true 
    */

    const setHeaderBtn = dsBridge.call("setHeaderTextBtn",{
        disabled:true,
        name:'分享',
        color:'#000',
        index:0
    },function(){
        console.log('按钮被点击')
    });

    // response
    /* 
        data:{}
    */




    // 移除
    /* 
        name:'removeHeaderTextBtn',
        params:{
            name:String|required,
        },
        async:false 
    */

    const response = dsBridge.call("removeHeaderTextBtn",{
        name:'分享'
    });

    // response
    /* 
        data:{}
    */
```

#### 5，分享

**实现功能：**  
```
    1，分享分为两种，一种正常分享链接，一种分享图片，根据分享的类型直接跳转对应的应用进行分享 
    2，在分享成功后进行回调，传入具体渠道信息
    3，js会传入具体的分享信息，需要原生进行配置
    4，分享的类型分为一下几种：qq,qzone,wechat(微信),moments(朋友圈),weibo,
```

**js调用方法：**  
``` javascript
    /* 
        name:'share',
        params:{
            type:String|required,
            title:String|required,
            content:String|required,
            url:String|required,
            picUrl:String|required,
        },
        async:true 
    */

    dsBridge.call("share",{
        type:'wechat',
        title:'hello',
        content:'world',
        url:'http://www.baidu.com',
        picUrl:'http://avatars.aibandata.com/avatar_1542593121.png'
    },funciton(response){

    });

    // response
    /* 
        data:{
            shared:true||false
        }
    */




   /* 
        分享图片时，图片数据为两种，一种线上url，一种base64位图片数据，dataType:'url'|'base64'
        name:'shareImage',
        params:{
            type:String|required,
            dataType:String|required,
            data:String
        },
        async:true 
    */

    dsBridge.call("shareImage",{
        type:'wechat',
        dataType:'base64',
        data:'data:image/png;base64,13244'
    },funciton(response){

    });

    // response
    /* 
        data:{
            shared:true||false
        }
    */
```

#### 6，设置header标题
**实现功能：**  
```
    1，直接切换header当前标题
```

**js调用方法：**  
``` javascript
    /* 
        name:'setHeaderTitle',
        params:{
            title:String|required
        },
        async:false 
    */

    const response = dsBridge.call("setHeaderTitle",{
        title:'爱扮'
    });

    // response
    /* 
        data:{}
    */
```

#### 7，复制文本到剪贴板
**实现功能：**  
```
    1，webview的长按可以复制文本，但是点击按钮用脚本复制内容有兼容问题，需要客户端提供方法,
    2，返回copyed值为true或false表示是否复制成功
```

**js调用方法：**  
``` javascript
    /* 
        name:'copyToClipboard',
        params:{
            content:String|required
        },
        async:false 
    */

    const response = dsBridge.call("copyToClipboard",{
        content:'hello world'
    });

    // response
    /* 
        data:{
            copyed:boolean
        }
    */
```

#### 8，保存图片到本地相册
**实现功能：**  
```
    1，将url地址的图片或者base64位图片保存到本地相册
    2，图片数据为两种，一种线上url，一种base64位图片数据，dataType:'url'|'base64'
    2，返回true或false表示是否复制成功
```

**js调用方法：**  
``` javascript
    /* 
        name:'savePicture',
        params:{
            dataType:String|required,
            data:String|required
        },
        async:true 
    */

    const response = dsBridge.call("savePicture",{
        dataType:'url',
        data:'http://avatars.aibandata.com/avatar_1542593121.png'
    },function(response){
        alert(response)
    });

    // response
    /* 
        data:{}
    */
```

#### 9，触发页面返回逻辑

**实现功能：**  
```
    1，触发headers上点击返回按钮的逻辑，执行返回逻辑
    2，返回到上个历史记录，无历史记录则返回到上一个webview或原生页面
```

**js调用方法：**  
``` javascript
    /* 
        name:'webviewBack',
        params:{
            appName:String|required
        },
        async:true 
    */

    const response = dsBridge.call("webviewBack");

    // response
    /* 
        data:{}
    */
```

#### 10，重置webview为全屏幕webview

**实现功能：**  
```
    1，webview覆盖整个手机屏幕,header导航栏初始化为透明背景，默认白色字体
    2，客户端监听页面滚动，在一个header高度的距离内，背景从透明渐变为白色，字体颜色从白色渐变为黑色。
    3，页面返回顶部时，在一个header高度的距离内，背景从白色渐变为透明，字体颜色从黑色渐变为白色。
    4，有设置header功能按钮时，渐变前后都保持按钮设置的颜色
```

**js调用方法：**  
``` javascript
    /* 
        name:'setFullWebviewStyle',
        params:{
            appName:String|required
        },
        async:true 
    */

    const response = dsBridge.call("setFullWebviewStyle"});

    // response
    /* 
        data:{
            ready:Boolean
        }
    */
```

## javascript 全局方法，提供给客户端调用


#### 1，返回事件处理

**实现功能：**  
```
    1，在页面执行返回逻辑时，判断有无此方法，有则优先执行此方法，无则正常执行返回逻辑
    2，等待此方法返回的状态决定是否执行返回逻辑，返回true则正常返回，返回false则取消返回
    3，无需传给javascript参数
```

**js调用方法：**  
``` javascript
    /* 
        name:'onWebviewBack',
        async:false 
    */

    dsBridge.register("onWebviewBack",function(){
        return {
            code:1,
            data:{
                ready:true||false
            },
            msg:'success'
        }
    });

    // response
    /* 
        data:{
            ready:Boolean,
        }
    */
```