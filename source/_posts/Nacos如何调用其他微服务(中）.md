---
title: Nacos：如何调用其他微服务（中）
date: 2021-3-18
tags:
	- Nacos
	- 微服务
categories:
	- SpringCloud
toc: true
---

上节我们将微服务模块注册到了注册中心Nacos中，接下来就是如何实现微服务模块之间接口的调用。

这里我们使用SpringCloud-Alibaba提供的OpenFeign进行微服务模块之间调用，如果微服务A需要调用微服务模块B的接口1，那么我们如何实现呢？

**原理**

​	微服务A、微服务B相互调用，前提是两个微服务都已经注册到注册中心Nacos中。然后微服务模块A1需要调用微服务B时，就去注册中心查看健康可用微服务模块B的地址，<!--more-->然后通过Nacos返回一个健康可用的微服务模块B1的地址给微服务模块A1，然后微服务模块通过利用OpenFeign向从Nacos中获得的地址发送**HTTP请求**给微服务模块B1的接口1，然后微服务A1调用微服务B1的接口1完成了。

<div align=center><img src="1615645734755-1b3246df-3e91-474b-b901-bd58ac654797-20210318224006023.jpeg" width="50%" height="50%" align=center/></div>

上面介绍了后面的一些原理后，现在我们实现微服务A调用微服务B的功能，这里以用户服务调用优惠券服务查看该用户的优惠券为例。

1.首先两个模块都要注册到注册中心中，上节已经实现

2.优惠券服务模块编写被调用接口

```java
@RestController
@RequestMapping("coupon/coupon")
public class CouponController {
    @Autowired
    private CouponService couponService;

    @RequestMapping(value="/member/list")
    public R membercoupons(){
        CouponEntity couponEntity = new CouponEntity();
        couponEntity.setCouponName("满100减10");
        return R.ok().put("coupons", Arrays.asList(couponEntity));
    }
}
```

3.用户服务模块引入OpenFeign依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
```

4.用户模块编写一个对应优惠券服务的声明式远程调用接口，指明需要调用微服务的名称以及调用该服务的哪个接口

​     OpenFeign通过声明式远程调用接口来实现调用服务，这样微服务A通过该接口就可以实现无感调用

```java
package com.lookstarry.doermail.member.feign;

import com.lookstarry.common.utils.R;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * @PackageName:com.lookstarry.doermail.member.feign
 * @NAME:CouponFeignService
 * @Description:
 * @author: yizhichangyuan
 * @date:2021/3/13 21:46
 */

/**
 * 这是一个声明式的远程调用，指明调用注册中心的那个服务的那个接口方法
 */
@FeignClient("doermail-coupon") //调用的微服务名称
public interface CouponFeignService {
    @RequestMapping("/coupon/coupon/member/list") //对应调用哪个接口
    public R membercoupons();
}
```

5.用户服务模块需要加上EnableFeignClients开启远程调用功能，指明自己哪个包是远程调用模块

```java
package com.lookstarry.doermail.member;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

/**
 * 1、调用远程调用别人的服务
 * 1）、引入open-feign
 * 2)、编写一个接口，告诉springCloud这个接口需要调用远程服务
 *  1、声明接口的每一个方法都是调用哪个远程服务的哪个请求
 * 3）、开启远程调用功能
 *
 */
@EnableFeignClients(basePackages = "com.lookstarry.doermail.member.feign")
@SpringBootApplication
@EnableDiscoveryClient
public class DoermailMemberApplication {

    public static void main(String[] args) {
        SpringApplication.run(DoermailMemberApplication.class, args);
    }
}
```