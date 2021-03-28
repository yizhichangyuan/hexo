---
title: Zxing二维码生成
date: 2021-2-23
tags:
	- Zxing
	- 短链
categories:
	- Spring
toc: true
---

> 二维码其实就是一个比特的矩阵，这个比特矩阵存储的是一个url，当二维码扫描时就会自动打开其中存储的url给服务器，这个url可能会通过GET方式给服务器传递一个扫码者的信息，服务器接收到请求后，将GET信息获取到进行处理就会返回一些页面给扫码者



### 背景需求

在项目中，店铺授权二维码为扫码者申请扫码权限，需要首先获取到扫码者的个人信息，才能进行授权，为了获取扫码者的个人信息，这里我们使用微信测试号回传用户信息的url，如下所示


```http
https://open.weixin.qq.com/connect/oauth2/authorize?appid=wx6d9e79f7e8844954&redirect_uri=http://81.70.249.115/o2o/shopadmin/addshopauthmap&role_type=1&response_type=code&scope=snsapi_userinfo&state=2#wechat_redirect
```

- appid字段：是申请微信测试号给予的appid开发者字段以及一个密匙secret（后面在获取用户信息会使用secret字段进行校验）
- redirect_uri字段：就是我们web应用接收微信回传个人信息的url
- state字段：可以自定义一些业务逻辑需要的一些信息，例如店铺授权需要知道往哪个店铺进行授权，需要知道店铺id，例如知晓二维码是否过期，封装createTime二维码的创建时间

扫码者打开微信扫描二维码时，就会访问这个url，微信就会将扫码者的信息传回给指定的redirect_uri，这个uri就可以通过code、accessToken、openId来获取用户的信息（这其中会涉及到多次二次验证），我们通过获取openId与数据库中的表tb_wechat_auth中查找该openId是否已经注册过，如果未注册就抓取其信息进行注册。

<!--more-->



### 实践

<font size=4 color='orange'>1. 申请微信测试号（[https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index](https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index)）</font>

<div align='center'><img src="1613878321918-810b6f23-69ad-4817-a58a-16739f945cec.png" alt="image-20210129220709957" style="zoom:50%;" /></div>

为了使用微信回传到我们指定的域名，微信要求需要实现对我们的域名进行认证，具体方法就是在我们的应用编写一个controller层，将微信传给的验证信息进行排序再返回即可
**这里回传排序好的信息给微信的url需要和图中的URL字段保持一致**

```java
 /**
 * 微信用来验证网站是否和openId绑定的路由
 */
@Controller
@RequestMapping(value = "/wechat")
public class WechatController {
    private static Logger logger = LoggerFactory.getLogger(WechatController.class);	
	
    @RequestMapping(method = RequestMethod.GET)
    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        String signature = HttpServletRequestUtil.getString(request, "signature");
        String timestamp = HttpServletRequestUtil.getString(request, "timestamp");
        String nonce = HttpServletRequestUtil.getString(request, "nonce");
        // 随机字符串
        String echoStr = HttpServletRequestUtil.getString(request, "echostr");

        //通过对signature进行校验，如果成功则原封不动输出echoStr，失败则什么都不输出
        PrintWriter out = null;
        try {
            // 从response中接收到输出实例，是输出到响应流中的
            out = response.getWriter();
            if (SignUtil.checkSignature(signature, timestamp, nonce)) {
                logger.debug("weixin get success...");
                out.println(echoStr);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (out != null) {
                out.close();
            }
        }
    }
}

package com.imooc.o2o.util.wechat;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Arrays;

public class SignUtil {
    // 与微信开发者中设置的token要一致
    private static String token = "o2o";

    /**
     * 验证微信api发过来的签名，需要对三者进行排序，并拼接成一个字符串，然后进行sha1加密，然后与signature进行对比
     *
     * @param signature 微信加密签名
     * @param timestamp 时间戳
     * @param nonce     随机数
     * @return
     */
    public static boolean checkSignature(String signature, String timestamp, String nonce) {
        String[] arr = new String[]{token, timestamp, nonce};
        Arrays.sort(arr);
        StringBuilder stringBuilder = new StringBuilder();
        for (int i = 0; i < arr.length; i++) {
            stringBuilder.append(arr[i]);
        }
        MessageDigest md = null;
        String tempStr = "";
        try {
            md = MessageDigest.getInstance("SHA-1");
            // 将三个字符串拼接成一个并进行SHA-1加密
            byte[] digest = md.digest(stringBuilder.toString().getBytes());
            tempStr = byteToStr(digest);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }

        stringBuilder = null;
        // 将sha1加密后的字符串与微信发送的签名进行比较，标识该请求来源于微信
        return tempStr != null ? tempStr.equals(signature.toUpperCase()) : false;
    }

    /**
     * 将字节组转换为十六进制字符串
     *
     * @param byteArray
     * @return
     */
    private static String byteToStr(byte[] byteArray) {
        String strDigest = "";
        for (int i = 0; i < byteArray.length; i++) {
            strDigest += byteToHexStr(byteArray[i]);
        }
        return strDigest;
    }

    /**
     * 将字节转换为十六进制字符串
     *
     * @param mByte
     * @return
     */
    private static String byteToHexStr(byte mByte) {
        char[] Digit = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F'};
        char[] tempArr = new char[2];
        tempArr[0] = Digit[(mByte >>> 4) & 0X0F];
        tempArr[1] = Digit[mByte & 0X0F];

        String s = new String(tempArr);
        return s;
    }


}
```

<font size=4 color='orange'>2. 生成二维码</font>

生成二维码首先需要引入Google的开发包zxing
```java
		<!--二维码相关-->
		<!-- https://mvnrepository.com/artifact/com.google.zxing/javase -->
		<dependency>
			<groupId>com.google.zxing</groupId>
			<artifactId>javase</artifactId>
			<version>3.4.1</version>
		</dependency>
```


生成二维码的工具类代码，可以看出二维码是BitMatrix，二维码生成后以png输出到响应流response中
```java
    /**
     * 生成二维码的图片流，其中content就是我们塞入二维码的内容
     * @param content
     * @param resp
     * @return BitMatrix 是一个比特的矩阵
     */
    public static BitMatrix generateQRCodeStream(String content, HttpServletResponse resp){
        // 给相应添加头部信息，主要告诉浏览器返回的是图片流
        resp.setHeader("Cache-Control", "no-store"); // 不设定缓存，因为二维码会过期
        resp.setHeader("Pragma", "no-cache");
        resp.setDateHeader("Expires", 0);
        resp.setContentType("image/png"); // 告诉浏览器是文件流

        // 设置图片的文字编码和内边框距
        Map<EncodeHintType, Object> hints = new HashMap<EncodeHintType, Object>();
        hints.put(EncodeHintType.CHARACTER_SET, "UTF-8");
        hints.put(EncodeHintType.MARGIN, 0);
        BitMatrix bitMatrix;
        try{
            // 生成二维码，参数顺序分别为：编码内容（一般为url）、编码类型、生成图片宽度、生成图片宽度、设置参数
            bitMatrix = new MultiFormatWriter().encode(content, BarcodeFormat.QR_CODE, 300, 300, hints);
        }catch(WriterException e){
            e.printStackTrace();
            return null;
        }
        return bitMatrix;
    }
```



<font size=4 color='orange'>3. 生成二维码的controller层</font>

controller层将微信回传的url封到二维码中，其中url中的state字段可以为我们添加一些辅助信息（例如授权时需要知道往哪个店铺进行授权员工操作就需要知道店铺的id），当扫码者扫描时回传用户的信息到指定的redirect_url通过把事先定义好的state字段以get形式返回，以进行辅助操作。这里我们塞入了两个字段一个是shopId以及二维码的生成时间createTime，createTime字段可以帮助我们在回传信息时判断二维码是否超过我们指定的时间间隔

```java
    /**
     * 将微信回传用户信息的url放入到二维码中，并通过Response返回个前台填充到<img src/>中的src属性中
     * @param request
     * @param response
     */
    @RequestMapping(value="/generateqrcodeshopauth", method=RequestMethod.GET)
    @ResponseBody
    public void generateQRCodeShopAuth(HttpServletRequest request, HttpServletResponse response){
        // 附加上必要的店铺信息到微信回传的state中，这样才知道回传用户信息往哪个店铺添加
        Shop shop = (Shop)request.getSession().getAttribute("currentShop");
        if(shop != null && shop.getShopId() != null){
            // 获取当前时间戳，以保证二维码的时间有效性，防止他人滥用，精确到毫秒
            long timeStamp = System.currentTimeMillis();
            // 加上aaa是为了方便剥离相应的信息
            String content = "{aaashopIdaaa:" + shop.getShopId() + ",aaacreateTimeaaa:" + timeStamp + "}";
            try{
                // 将content的信息先进行Base64编码，防止特殊字符对url进行干扰
                String longUrl = urlPrefix + authUrl + urlMiddle + URLEncoder.encode(content, "UTF-8") + urlSuffix;
                // 将目标url转为短的URL，防止微信扫码失效
                String shortUrl = ShortNetAddressUtil.getShortURL(longUrl);
                BitMatrix bitMatrix = CodeUtil.generateQRCodeStream(shortUrl, response);
                // 将二维码图片流输出到响应流中
                MatrixToImageWriter.writeToStream(bitMatrix, "png", response.getOutputStream());
            }catch(Exception e){
                e.printStackTrace();
                logger.error(e.toString());
            }
        }
    }
```


前台通过<img>标签中的src就可以请求响应的二维码图片流
```html
<img src="/o2o/shopadmin/generateqrcodeshopauth" id="auth-img"/
```



<font size=4 color='orange'>4. 微信回传信息处理controller层</font>

扫码者扫码后回传信息处理的controller，这里需要和redirect_url保持一致，这样才能正确接收到微信回传的用户信息。

- 获取微信回传用户信息，主要是通过code字段、appid以及secret字段（这些字段是通过微信测试号进行获取的，获取需要绑定指定的域名的controller层，这个controller层主要是微信辅助校验域名的有效性）获取到用户的token字段以及openId字段，然后再通过token字段和openId字段回传给微信获取用户的信息
- 为了防止二维码泄漏，被反复操作，这里通过vaildQRCode来检测回传的state字段中的createTime检测二维码是否过期。
- 为了防止二维码被反复扫描，恶意入库，这里通过checkOpenIdRegister检测回传用户信息的openId是否已经注册过，如果已经注册过就不在入库。
```java
/**
     * 将二维码打开的url回传的微信用户信息进行授权
     * @param request
     * @return
     */
    // todo 调试
    @RequestMapping(value="/addshopauthmap")
    public ModelAndView addShopAuthMap(HttpServletRequest request){
        ModelAndView authFail = new ModelAndView("shop/shopauthfail");
        ModelAndView authSuccess = new ModelAndView("shop/shopauthsuccess");
        // 获取店铺信息
        String state = HttpServletRequestUtil.getString(request, "state");
        String jsonStr = state.replaceAll("aaa", "\"");
        ObjectMapper objectMapper = new ObjectMapper();
        WechatInfo wechatInfo = null;
        try{
            wechatInfo = objectMapper.readValue(jsonStr, WechatInfo.class);
        } catch (JsonMappingException e) {
            e.printStackTrace();
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }

        // 获取微信回传得到的用户信息
        String code = request.getParameter("code");
        PersonInfo personInfo = null;
        try{
           UserAccessToken userAccessToken =  WechatUtil.getUserAccessToken(code);
           String openId = userAccessToken.getOpenId();
           WechatUser wechatUser = WechatUtil.getUserInfo(userAccessToken.getAccessToken(), openId);
           personInfo = WechatUtil.getPersonInfo(wechatUser);
           personInfo.setUserType(4); // 设置为店家的管理员
           // 如果openId对应用户未注册过则进行注册
           if(!checkOpenIdRegister(openId)){
               WechatAuth wechatAuth = new WechatAuth();
               wechatAuth.setOpenId(openId);
               wechatAuth.setPersonInfo(personInfo);
               WechatExecution execution = wechatAuthService.register(wechatAuth);
               if(execution.getState() != WechatAuthStateEnum.SUCCESS.getState()){
                   logger.info("error");
                   return authFail;
               }
           }else{
               // 如果注册过，则查询数据库以填充personInfo的userId，后面才可以进行店铺授权
               WechatAuth temp = wechatAuthService.getWechatAuthById(openId);
               personInfo = temp.getPersonInfo();
           }
        }catch (Exception e){
            logger.error(e.toString());
            return authFail;
        }

        // 检查是否重复授权或者二维码过期，如果符合就返回失败的视图页面
        if(checkAlreadyAuth(wechatInfo.getShopId(), personInfo.getUserId()) || !vaildQRCode(wechatInfo)){
            return authFail;
        }

        // 如果前面操作都通过即未进行过授权以及二维码未过期，进行店铺授权
        if(wechatInfo != null && wechatInfo.getShopId() != null && personInfo != null && personInfo.getUserId() != null){
            try{
                ShopAuthMap shopAuthMap = new ShopAuthMap();
                shopAuthMap.setTitle("员工");
                shopAuthMap.setTitleFlag(1);
                Shop shop = new Shop();
                shop.setShopId(wechatInfo.getShopId());
                shopAuthMap.setShop(shop);
                shopAuthMap.setEmployee(personInfo);
                ShopAuthMapExecution shopAuthMapExecution = shopAuthMapService.addShopAuthMap(shopAuthMap);
                if(shopAuthMapExecution.getState() == ShopAuthMapStateEnum.AUTH_SUCCESS.getState()){
                    return authSuccess;
                }else{
                    logger.error(shopAuthMapExecution.getStateInfo());
                    return authFail;
                }
            }catch (Exception e){
                logger.error(e.toString());
                return authFail;
            }
        }else{
            return authFail;
        }
    }

    /**
     * 检查openId是否注册过
     * @param openId
     * @return
     */
    private Boolean checkOpenIdRegister(String openId){
        WechatAuth wechatAuth = wechatAuthService.getWechatAuthById(openId);
        if(wechatAuth != null && wechatAuth.getPersonInfo() != null){
            return true;
        }
        return false;
    }

    /**
     * 检查二维码是否过期
     * @param wechatInfo
     * @return
     */
    private Boolean vaildQRCode(WechatInfo wechatInfo) {
        if (wechatInfo != null && wechatInfo.getCreateTime() != null && wechatInfo.getShopId() != null) {
            Long expireTime = wechatInfo.getCreateTime();
            Long currentTime = System.currentTimeMillis();
            // 如果二维码创建时间和现在时间超过10分钟则失效
            if (currentTime - expireTime > 600000) {
                return false;
            } else {
                return true;
            }
        } else {
            return false;
        }
    }

    /**
     * 检查是否已经授权过，避免重复授权数据库一致插入
     * @param shopId
     * @param employeeId
     * @return
     */
    private Boolean checkAlreadyAuth(Long shopId, Long employeeId){
        ShopAuthMapExecution shopAuthMapExecution = shopAuthMapService.listShopAuthMapByShopId(shopId, 0, 100);
        List<ShopAuthMap> list = shopAuthMapExecution.getShopAuthMapList();
        for(ShopAuthMap shopAuthMap : list){
            if(shopAuthMap.getEmployee().getUserId() == employeeId){
                return true;
            }
        }
        return false;
    }
```
