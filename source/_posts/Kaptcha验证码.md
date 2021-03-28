---
title: Kaptcha验证码
date: 2021-1-16
tags:
	- Kaptcha
categories:
	- Spring
toc: true
---
Kaptcha是java中一个验证码图片校对的组件，其核心是通过配置一个Servlet为前端提供验证码图片，并将对应图片正确验证码放入到session中，便于校对用户提交的验证码。

###  <font size=4 color='orange'>1. 具体步骤</font>

1. 首先引入pom
2. web.xml配置Kaptcha的Servlet，其中主要指定验证码图片的宽度、高度、几个字符、字体、字体颜色、干扰线的颜色以及相应请求验证码图片的url-pattern
  <!--more-->
3. 前端img的src属性填充的url就为url-pattern，img会向Kaptcha的Servlet请求验证码图片
4. 前端表单提交的时候，同时需要把用户输入的验证码也放入到表单中，以便发给后端以便进行比对
5. 后端在接收前端表单数据，第一步应该是检查验证码是否正确，如何检查？Kaptcha在向前端发送验证码图片的同时，就已经把验证码图片对应的正确的验证码放入的session中，所以后端只需要比对request中传来表单中验证码数据和Session中获取的数据是否一致即可，不一致就直接返回错误信息（其中Session的参数key是`com.google.code.kaptcha.Constants.KAPTCHA_SESSION_KEY`





#### <font size=4 color='orange'>2. 实践</font>

1.首先在pom.xml中引入依赖

```xml
    <!--验证码校验-->
    <!-- https://mvnrepository.com/artifact/com.github.penggle/kaptcha -->
    <dependency>
      <groupId>com.github.penggle</groupId>
      <artifactId>kaptcha</artifactId>
      <version>2.3.2</version>
    </dependency>
```
2.web.xml配置Kaptcha的Servlet，其中主要指定验证码图片的宽度、高度、几个字符、字体、字体颜色、干扰线的颜色以及相应请求验证码图片的url-pattern

```xml
    <!--Kaptcha验证码servlet-->
    <servlet>
        <servlet-name>Kaptcha</servlet-name>
        <servlet-class>com.google.code.kaptcha.servlet.KaptchaServlet</servlet-class>
        <!--是否有边框-->
        <init-param>
            <param-name>kaptcha.border</param-name>
            <param-value>no</param-value>
        </init-param>
        <!--字体颜色-->
        <init-param>
            <param-name>kaptcha.textproducer.font.color</param-name>
            <param-value>red</param-value>
        </init-param>
        <!--图片宽度-->
        <init-param>
            <param-name>kaptcha.image.width</param-name>
            <param-value>135</param-value>
        </init-param>
        <!--使用哪些字符来生成验证码-->
        <init-param>
            <param-name>kaptcha.textproducer.char.string</param-name>
            <param-value>ACDEFHKPRSTWX345679</param-value>
        </init-param>
        <!--图片高度-->
        <init-param>
            <param-name>kaptcha.image.height</param-name>
            <param-value>50</param-value>
        </init-param>
        <!--字体大小-->
        <init-param>
            <param-name>kaptcha.textproducer.font.size</param-name>
            <param-value>43</param-value>
        </init-param>
        <!--干扰线的颜色-->
        <init-param>
            <param-name>kaptcha.noise.color</param-name>
            <param-value>black</param-value>
        </init-param>
        <!--字符个数-->
        <init-param>
            <param-name>kaptcha.textproducer.char.length</param-name>
            <param-value>4</param-value>
        </init-param>
        <!--字体-->
        <init-param>
            <param-name>kaptcha.textproducer.font.names</param-name>
            <param-value>Arial</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>Kaptcha</servlet-name>
        <url-pattern>/Kaptcha</url-pattern>
    </servlet-mapping>
```
3.前端img的src属性填充的url就为url-pattern，img会向Kaptcha的Servlet请求验证码图片
```html
<div class="item-title label">验证码</div>
<div class="item-input">
<input type="text" id="j_kaptcha" placeholder="请输入验证码"/>
// src的地址和servlet的地址一致，这里的o2o是项目的根目录
<img id="kaptcha-img" src="/o2o/Kaptcha" title="点击更换图片" alt="点击更换图片" onclick="changeVerifyCode(this)">
```
```javascript
// 点击切换图片就是再次发送同一个请求，加上一个两位的数字，就会重新更换前端的验证码图片
function changeVerifyCode(img){
    img.src = "/o2o/Kaptcha?" + Math.floor(Math.random() * 100);
}
```
<div align='center'><img src="image-20210129220709957.png" alt="image-20210129220709957" style="zoom:50%;" /></div>

4.前端表单提交的时候，同时需要把用户输入的验证码也放入到表单中，以便发给后端以便进行比对

```javascript
// 获取用户输入的验证码
var fromData = new FormData();
var verifyCode = $("#j_kaptcha").val();
if (!verifyCode) {
  $.toast("验证码为空");
  return;
}
formData.append("verifyCodeActual", verifyCode);
$.ajax({
  url: registerShopURL,
  type: 'POST',
  data: formData,
  contentType: false, // 因为有文字和图片，所以为false
  processData: false,
  cache: false,
  success: function (data) {
    if (data.success) {
      $.toast("提交成功！");
    } else {
      $.toast("提交失败！" + data.errMsg);
    }
    // 不管提交是否成功，都要改变一下验证码，因为点击事件连接着一个切换图片的事件
    $('#kaptcha-img').click();
  }
});
```
5.后端在接收前端表单数据，第一步应该是检查验证码是否正确，如何检查？Kaptcha在向前端发送验证码图片的同时，就已经把验证码图片对应的正确的验证码放入的session中，所以后端只需要比对request中传来表单中验证码数据和Session中获取的数据是否一致即可，不一致就直接返回错误信息（其中Session的参数key是`com.google.code.kaptcha.Constants.KAPTCHA_SESSION_KEY`）

```java
package com.imooc.o2o.util;

import com.google.code.kaptcha.Constants;
import javax.servlet.http.HttpServletRequest;

public class CodeUtil {
    public static Boolean checkVerifyCode(HttpServletRequest request){
        // 在shopOperation.html中img的src发送请求给/o2o/Kaptcha给Kaptcha Servlet的时候，
        // 同时图片真实的验证码就会种植在session中，便于校验用户输入与真实是否相同
        String verifyCodeExpected = (String)request.getSession().getAttribute(Constants.KAPTCHA_SESSION_KEY);
        System.out.println("verifyCodeExpected:" + verifyCodeExpected);
        String verifyCode = HttpServletRequestUtil.getString(request, "verifyCodeActual");
        System.out.println("verifyCodeActual:" + verifyCode);
        if(verifyCode == null || !verifyCode.equals(verifyCodeExpected)){
            return false;
        }
        return true;
    }
}
```
