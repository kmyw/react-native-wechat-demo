### 版本
```javascript
react-native 0.61.2
react-native-wechat 1.9.12
```
### 安装配置
- react-native-wechat 安装
```javascript
yarn add react-native-wechat
// rn 版本 < 0.60
react-native link react-native-wechat
// rn 版本 > 0.60
cd /ios/
pod install
```
### Android 配置

- 在/android/app/src包名目录下创建wxapi包，创建WXEntryActivity.java和WXPayEntryActivity.java
![image](https://user-gold-cdn.xitu.io/2019/11/1/16e2664efb1d5db2?w=339&h=572&f=png&s=43894)
```java
// WXEntryActivity.java
package com.xxx.wxapi;

import android.app.Activity;
import android.content.Intent;
import android.os.Bundle;

import com.theweflex.react.WeChatModule;
import com.wechatusageexample.MainActivity;

public class WXEntryActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        if (MainActivity.isActivityCreated) {
            WeChatModule.handleIntent(this.getIntent());
        } else {
            // 如果应用未在后台启动，就打开应用
            Intent intent = new Intent(getApplicationContext(), MainActivity.class);
            intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
            startActivity(intent);
        }

        finish();
    }
}
```

```java
// WXPayEntryActivity.java
package com.xxx.wxapi;

import android.app.Activity;
import android.os.Bundle;

import com.theweflex.react.WeChatModule;

public class WXPayEntryActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        WeChatModule.handleIntent(getIntent());
        finish();
    }
}
```

- 配置 AndroidManifest.xml
![image](https://user-gold-cdn.xitu.io/2019/11/1/16e2666b6852a84f?w=1162&h=337&f=png&s=79788)
```xml
<activity
    android:name=".wxapi.WXEntryActivity"
    android:label="@string/app_name"
    android:exported="true"
/>
<activity
    android:name=".wxapi.WXPayEntryActivity"
    android:label="@string/app_name"
    android:exported="true"
/>
```

- 在proguard-rules.pro中添加混淆忽略代码

```
-keep class com.tencent.mm.sdk.** {
  *;
}
```

### iOS配置
- link 库
![image](https://user-gold-cdn.xitu.io/2019/11/1/16e2666e90b10837?w=1311&h=715&f=png&s=89604)
```
#Build Phases ➜ Link Binary With Libraries
SystemConfiguration.framework
CoreTelephony.framework
libsqlite3.0
libc++
libz
```
    
- 在AppDelegate.m中添加代码
    
    
```
#import <React/RCTLinkingManager.h>

#pragma mark - Handle URL

- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url
  sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
{
  return [RCTLinkingManager application:application openURL:url
   sourceApplication:sourceApplication annotation:annotation];
}

// ios 9.0+
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url
    options:(NSDictionary<NSString*, id> *)options
{
  return [RCTLinkingManager application:application openURL:url options:options];
}
```
    
 - 配置白名单
  
   在 ios/Info.plist 添加
```xml
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>weixin</string>
  <string>wechat</string>
</array>
```
 - 配置URL Schema
  
![image](https://user-gold-cdn.xitu.io/2019/11/1/16e266772ac03154?w=1657&h=648&f=png&s=65059)
   打开Xcode 选择Targets > info，配置URL Types
 - 配置Universal Link（ios需要配置Universal Link，才能连接打开app）
 ![image](https://user-gold-cdn.xitu.io/2019/11/1/16e2668226247b59?w=1882&h=630&f=png&s=81723)
   - 创建一个文件名是apple-app-site-association的文件, 这个文件需要上传到自己服务器，以下面举例子，这里的appid是prefix + bundle id, path是下载路径，比如自己的服务器网站域名是，https://xx.xxx.com则访问https://xx.xxx.com/download可以下载到apple-app-site-association文件就算ok, 然后打开打开Xcode 选择Targets > Capabilities，打开Assoication Domains选项，添加一项：applinks:xxx.xxx.com (自己服务器域名)
    ```
    {
        "applinks": {
            "apps": [],
            "details": [
                {
                    "appID": "8NL2H3YTL7.org.reactjs.native.example.dummpAppJacky",
                    "paths": [ "/download" ]
                }
            ] 
        }
    }
    ```
    
### 调用
- 微信参数说明
```
const wxAppId = ""; // 微信开放平台注册的app id
const wxAppSecret = ""; // 微信开放平台注册得到的app secret
const wxMerchantId = ""; // 微信商户ID
const wxTransSecret = ""; // 商户api秘钥
```
- 初始化
```javascript
import * as WeChat from 'react-native-wechat';

WeChat.registerApp(wxAppId);
```
- 登录
```javascript
wechatLogin() {
    if (!this.state.isWXInstalled) {
      this.showAlert('微信未安装');
      return;
    }
    WeChat.sendAuthRequest('snsapi_userinfo', 'wechat_sdk_demo').then((response) => {
      this.getOpenId(response.code);
    }).catch((error) => {
      let errorCode = Number(error.code);
      if (errorCode === -2) {
        this.showAlert('已取消授权登录'); // errorCode = -2 表示用户主动取消的情况，下同
      } else {
        this.showAlert('微信授权登录失败');
      }
    })
  }
  
  /// 获取openId
  getOpenId(code) {
    this.progressHUD.show();
    let requestUrl = "https://api.weixin.qq.com/sns/oauth2/access_token?appid=" + wxAppId + "&secret=" + wxAppSecret + "&code=" + code + "&grant_type=authorization_code";
    fetch(requestUrl).then((response) => response.json()).then((json) => {
      console.log('获取微信openid成功');
      console.log(json);
      this.getUnionId(json.access_token, json.openid);
    }).catch((error) => {
      this.progressHUD.hide();
      this.showAlert('微信授权登录失败');
    })
  }
  
  getUnionId(accessToken, openId) {
    let requestUrl = "https://api.weixin.qq.com/sns/userinfo?access_token=" + accessToken + "&openid=" + openId;
    fetch(requestUrl).then((response) => response.json()).then((json) => {
      console.log('获取微信unionid成功');
      console.log(json);
      // TODO: 这里openId和unionId都已经成功获取了，调用用户自己的接口传递openId或unionId登录或注册
      // put your login method here...
    }).catch((error) => {
      this.progressHUD.hide();
      this.showAlert('微信授权登录失败');
    })
  }
```
- 分享
```javascript
// 分享到好友
  shareToFriend() {
    if (!this.state.isWXInstalled) {
      this.showAlert('微信未安装');
      return;
    }
    WeChat.shareToSession({
      type:'news',
      webpageUrl:'https://www.baidu.com',
      title:'Test sharing',
      description:'This is a test'
    }).then((response) => {
      console.log(response);
      this.showAlert('分享成功');
    }).catch((error) => {
      let errorCode = Number(error.code);
      if (errorCode === -2) {
        this.showAlert('分享已取消');
      } else {
        this.showAlert('分享失败');
      }
    })
  }
  
  // 分享到朋友圈
  shareToTimeline() {
    if (!this.state.isWXInstalled) {
      this.showAlert('微信未安装');
      return;
    }
    WeChat.shareToTimeline({
      type:'news',
      webpageUrl:'https://www.baidu.com',
      title:'Test sharing',
      description:'This is a test'
    }).then((response) => {
      console.log(response);
      this.showAlert('分享成功');
      
    }).catch((error) => {
      let errorCode = Number(error.code);
      if (errorCode === -2) {
        this.showAlert('分享已取消');
      } else {
        this.showAlert('分享失败');
      }
    })
  }
```
- 支付
```javascript
// 支付
  wechatPay() {
    if (!this.state.isWXInstalled) {
      this.showAlert('微信未安装');
      return;
    }
    /**************添加支付处理过程****************/
    
    // 第一步，获取预订单prepayId，生成预订单最好让做后台接口的童鞋来完成，app端调用接口获取预订单
    let prepayId = ""; // 这里预订单是从接口获取的，这里简写仅做演示
    let tempTime = Date.parse(new Date());
    let timestamp = (tempTime/1000).toString();
    let nonce_str = MD5.hexMD5(timestamp);
  
    // 第二步，拼装参数
    let params = {
      "appid": wxAppId,
      "noncestr": nonce_str,
      "package": "Sign=WXPay",
      "partnerid": wxMerchantId,
      "timestamp": timestamp,
      "prepayid": prepayId,
    };
    let paramsList = [];
    let sortedKeys = Object.keys(params).sort();
    for (let i = 0; i < sortedKeys.length; i++) {
      let keyValueCombo = sortedKeys[i] + "=" + params[sortedKeys[i]];
      paramsList.push(keyValueCombo);
    }
    let paramsString = paramsList.join('&');
    let finalString = paramsString + "&key=" + wxTransSecret;
    let encryptedStr = MD5.hexMD5(finalString).toUpperCase(); // MD5加密后转为大写
  
    // 第三步，调起微信客户端支付
    WeChat.pay({
      appId: wxAppId,
      partnerId: wxMerchantId,
      prepayId: prepayId,
      nonceStr: nonce_str,
      timeStamp: timestamp,
      package: 'Sign=WXPay',
      sign: encryptedStr
    }).then((response) => {
      console.log('支付成功');
      console.log(response);
      let errorCode = Number(response.errCode);
      if (errorCode === 0) {
        this.showAlert('支付成功');
        // TODO: 这里处理支付成功后的业务逻辑，比如支付成功跳转页面、清空购物车。。。。
        // .....
      } else {
        this.showAlert(response.errStr);
      }
    }).catch((error) => {
      let errorCode = Number(error.code);
      if (errorCode === -2) {
        this.showAlert('已取消支付');
      } else {
        this.showAlert('支付失败');
      }
    });
  }
```

### 参考
- [react-native-wechat](https://github.com/yorkie/react-native-wechat)
- [react-native-blog-examples](https://github.com/mrarronz/react-native-blog-examples/blob/master/Chapter14-Wechat_Login_Share_Pay/WechatUsageExample/ExampleNote.md)
