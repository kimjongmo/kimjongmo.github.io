---
layout: post
title: 스프링부트 HTTP요청을 HTTPS 둘다 사용하기
data: 2019-05-29 18:05:00
categories: spring
permalink: /spring/spring-boot-redirect-http-to-https
tags: spring-boot https
---

# 스프링부트 HTTPS 적용

[스프링 부트 HTTPS 적용시키기](https://kimjongmo.github.io/spring/spring-boot-https)

프로젝트에 자가 서명을 이용한 후 http요청을 하게 되면 아래와 같은 응답을 받게된다. 

![](/img/18-05-29/bad_request.PNG)

당연히도 해당 포트는 이제 더이상 http요청을 받지않도록 설정이 되있기 때문이다. http요청도 가능하게 하는 방법은 아래와 같다.

## 멀티 커넥터 

[참고 : <https://drissamri.be/blog/java/enable-https-in-spring-boot/> ]

http요청은 8080포트로 https요청은 8443포트로 이용가능하도록 만든다. 전에 디폴트 커넥터를 https용도로 만들었기 때문에 새로 생성하는 커넥터는 http용도로 만들어 추가한다.

```java
@Configuration
public class ConnectorConfig {

    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
        tomcat.addAdditionalTomcatConnectors(createSslConnector());
        return tomcat;
    }

    private Connector createSslConnector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setScheme("http");
        connector.setSecure(false);
        connector.setPort(8080);
        return connector;
    }

}
```



## HTTP요청을 HTTPS으로 리다이렉트시키기

현재 두개의 커넥터를 이용하여 http, https를 모두 이용할 수 있도록 만들었지만 가능하다면 모든 요청은 https로 가는 것이 좋다고 위의 참고글에서 설명하는 것 같다. 이것을 가능하게 하기 위해서는 아래와 같이 redirect설정과 `TomcatEmbeddedServletContainerFacotry`의 `postProcessContext`메소드를 오버라이딩하여 가능하다.

```java
@Configuration
public class ConnectorConfig {

    @Bean
    public ServletWebServerFactory servletContainer() {

        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory(){
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                securityConstraint.addCollection(collection);
                context.addConstraint(securityConstraint);
            }
        };
        tomcat.addAdditionalTomcatConnectors(createSslConnector());
        return tomcat;
    }

    private Connector createSslConnector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setScheme("http");
        connector.setSecure(false);
        connector.setPort(8080);
        connector.setRedirectPort(8443);
        return connector;
    }

}
```

이와 같이 준비가 되었다면 http의 8080포트로 접속해본다.



## 특정 경로에 대해서만 https로 리다이렉트시키기

모든 HTTP에 대한 요청을 HTTPS로 바꾸는 것보다 특정한 경로에 대해서만 HTTPS로 전환하고 싶다면 위의 `SecurityConstraint`에 패턴을 변경한다. 예를 들어서 `/test` 경로만 HTTPS로 전환시키고 싶다면 아래와 같이 변경한다.

```java
...

collection.addPattern("/test");

...
```



