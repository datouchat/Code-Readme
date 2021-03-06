
#接口定义
接口数据获取---->客户端与服务端交互 均为proto作为数据序列化传输

1.结构如下



《------------Get-----------------》
```
byte[] bytes = getFromBody;
byte[] decodeBytes = Base64.decode(bytes);
Net.TransportData data = Net.TransportData.parseFrom(decodeBytes);
Map<String, String> header = data.getHeaderMap(); //Header头，使用该Header头，而不是Http中的Header
String action = data.getAction();//action，根据此Action来unpack
Any any = data.getBody();
if(action.equals("api.push.auth")){
  ApiPushAuth.ApiPushAuthRequest request = any.unpack(ApiPushAuth.ApiPushAuthRequest.class);
}else if(action.equals("api.site.mute")){
  ApiSiteMute.ApiSiteMuteRequest  request = any.unpack(ApiSiteMute.ApiSiteMuteRequest.class);
}
``````



<----------业务处理-------->

《------------return-------------》

```
Map<String, String> header =new HashMap<String, String>();
header.put("_"+Net.TransportDataHeaderKey.HeaderErrorCode_VALUE, "success");//这次请求成功还是失败
header.put("_"+Net.TransportDataHeaderKey.HeaderErrorInfo_VALUE, "");

ApiPushAuth.ApiPushAuthResponse response = ApiPushAuthResponse.newBuilder()
				.setSiteIsMute(boolean)
				.build();
        
Net.TransportData data = Net.TransportData.newBuilder()
				.setAction("api.push.auth")                 ------------->action一定要和Body中的Any.pack的类型一致！！！
				.putAllHeader(header)
				.setBody(Any.pack(response))
				.build();
byte[] bytes = Base64.encodeBase64(data.toByteArray());

输出bytes   
```


----------------Finish-------------




一、http://host:port/?action=api.push.auth&body_format=pb
1.该接口在客户端登录后以及每次启动均会调用，服务端获取用户推送token后需入库。可另做用户活跃度判断标准
2.数据结构
```
ApiPushAuthRequest {
    string sitePubkPem;         //站点生成的公钥Base64后的值 
    string siteAddressApi;      //站点Api地址（站点地址）
    string siteName;            //站点的名称
    string pushToken;           //push服务的token，由推送平台在客户端生成
    PushTokenType pushTokenType;  //设备类型，由服务端判断选择何种推送平台推送消息
    string devicePubkPem;         //客户端初次运行生成的公钥，可做设备唯一性校验
    string timestampSeconds;    // = string(timestamp unit: seconds)
    string signTimestampBase64;    // sign(timestampSeconds)---->客户端私钥加密后的时间
    string signSitePubkPemBase64;    // sign(sitePubkPem)--->客户端私钥加密后的站点公钥
    string siteUserId;              //用户的userId
}
```

```
message ApiPushAuthResponse {
    bool siteIsMute = 1;        //当前站点是否静音
}
```

```
enum PushTokenType {
    pushTokenInvalid   = 0;
    pushTokenIOS       = 1;
    pushTokenAndroid   = 2;
    pushTokenXiaoMi    = 3;
    pushTokenHuawei    = 4;
    pushTokenGCM       = 5;
}
```


二、http://host:port/?action=api.push.cancel&body_format=pb
1.该接口由客户端退出登录后调用，一般视作客户端切换账号（推送token不变，但是对应的用户更换）
2.数据结构
```
message ApiPushCancelRequest {
    string sitePubkPem        = 1;   //站点公钥
    string devicePubkPem      = 2;  //设备公钥
    string timestampSeconds   = 3; //单位s
    string signTimestamp      = 4;  //客户端私钥加密后的时间
}
```
```
message ApiPushCancelResponse {}
```

三、http://host:port/?action=api.site.mute&body_format=pb
1.该接口由客户端开启或关闭静音调用，服务端根据此判断是否给用户推送或者推送是否带声音
2.数据结构
```
message ApiSiteMuteRequest {
    string siteCode = 1;        //code = sha1(pubkPem) 站点公钥sha1值
    string siteUserId = 2;      //site userId
    bool isMute = 3;            //is mute value
}
```
```
message ApiSiteMuteResponse {

}
```
四、http://host:port/?action=api.push.notification&body_format=pb
1.由站点申请推送消息--->推送服务端接受---->推送
2.数据结构

```
message ApiPushNotificationRequest {
    PushHeader pushHeader = 1;
    PushBody pushBody = 2;
}
```

```
message ApiPushNotificationResponse {
}
```

```
message PushHeader {
    string siteAddress = 1;
    string siteName = 2;

    string sitePubkPemId    = 3;
    string timestampSeconds = 4; // string(timestamp unit: seconds)
    string signTimestamp    = 5;
}
```

```
//support batch send
message PushBody{
    // subTitle
    // if roomId != "" && roomName != "", show the subTitle  房间name（群名称或者对方名称）可做推送的SubTitle
    string roomId   = 1;
    string roomName = 2;
    core.MessageRoomType roomType = 3;
    // Sender Info
    // if fromUserId != "" && fromUserName != "", show the sender info
    string fromUserId   = 4;
    string fromUserName = 5;
    // Content
    // if pushContent != "", use pushContent
    // else make the content from msgType.
    core.MessageType msgType = 6;
    string pushContent       = 7;  必须在站点设置选择显示消息内容，否则推送不显示内容

    // icon
    // avatar must be prefix with "http://" or "https://" 
    string iconHref = 8;

    // GotoUrl
    // if gotoUrl == "", make the url from the infomation above.
    string gotoUrl = 9;

    // pemId = sha1(pem)，因为平台这边有pem，同时不需要做sign的verify，所以传递Ids，以减少请求包大小
    repeated string toDevicePubkPemIds   = 10;  //repeated 在java中就是List

    string msgId = 11;//push message id
}
```

```
enum MessageRoomType {
    MessageRoomGroup = 0;
    MessageRoomU2    = 1;
}
```

```
enum MessageType {
    MessageInvalid = 0;
    MessageNotice  = 1;
    MessageText    = 2;
    MessageImage   = 3;
    MessageAudio   = 4;
    MessageWeb     = 5;
    MessageWebNotice = 6;
    MessageDocument = 7;
    MessageVideo = 8;
    MessageRecall = 9;

    // reverts 5-19

    // event message start
    MessageEventFriendRequest   = 20;
    MessageEventStatus          = 21;   // NoDB    -> StatusMessage
    MessageEventSyncEnd    = 22;        // NoDB    -> there's no new message in server.
}
```




















# 客户端（Android）

### 主要以app Moudle为主，编译也是编译该Moudle，推荐使用Android studio 3.0版本

##### 一、资源文件
1.站点连接地址 ---->res/values(values-zh-rCN)/strings.xml  zh-rCn下为中国地区访问资源文件，泛指系统语言设置为中文情况下。

2.appLogo---->@mipmap/ic_duck_chat_logo   @mipmap/ic_duck_chat_logo_round  round为安卓8.0以上系统默认采用的圆形图标  注意logo分辨率适配

3.开屏页面--->@mipmap/ic_duck_chat_splash  

4.应用包名修改--->app Moudle下build.gradle文件下applicationId

5.应用版本号 projece目录下 gradle.propertiesAPPLICATION_VERSION_NAME 例如1.1.1   APPLICATION_VERSION_CODE 必须为数字例如11

6.推送渠道appid ---->友盟 com.akaxin.zaly.bean.Constants 类下  UM_PUSH_SECRET  UM_APP_KEY 2个值

		     小米 com.akaxin.zaly.bean.Constants 类下 MI_APP_ID  MI_APP_KEY
		     
		     华为 app Moudle下build.gradle文件下 hwpush
		     
7.平台地址（推送服务器） com.akaxin.zaly.bean.Constants 类下  PLATFORM_ADDRESS

##### 二、编译
Android studio ---》Build-》Generate Signed Apk

详情 https://www.cnblogs.com/gao-chun/p/4891275.html 注意 Signature Versions V1(Jar Signature) V2(Full APK Signature) 都选上







# 客户端（iOS）

### 使用cocoapods  需要在Mac上安装cocoapods（谷歌百度都有）然后拉下项目执行pod install

##### 一、资源文件
1.站点连接地址 ——> config -> baseconfig.swift

2.appLogo ——> images.xcassets -> appicon。删除原来的 自己替换 https://icon.wuruihong.com/  这个网址可以快速制作

3.开屏页面—>   images.xcassets -> main -> 启动图 3张自己替换

4.应用包名修改--->项目打开后的首级目录。bundle identifier

5.应用版本号  项目打开后的首级目录。version

6.推送渠道appid ——>  zaly -> appdelegate+push.swift.     didRegisterForRemoteNotificationsWithDeviceToken.   已经答应了devicetoken的string类型 （打包前需要设置推送证书，参考：https://www.jianshu.com/p/adfce0921c09）

7.平台地址（推送服务器） config -> baseconfig.swift
  如：http://127.0.0.1:9001/push-platform/
8、在电脑安装cocoapods 
   然后pod update
   不然消息收到不会提示
