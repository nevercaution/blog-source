---
title: "spring boot 에서 jsp view 만들기 (feat freemarker)"
catalog: true
date: 2018-02-22 11:33:19
subtitle:
header-img: "img/header_img/bg.png"
tags:
- springboot
- jsp
- freemarker
- gradle
---

spring 을 사용하다가 spring boot 로 넘어오면서 front, back 을 나누어서 백단은 나름 Restful 하게 해서 api 콜만 처리하는 방식으로 변경하는 중이다.(front 는 react로 구성하는 중이다.) 그래서 spring boot 에서는 따로 view 처리해야할 일이 없었는데 기존에 spring 에서 view 처리를 해주는 요청을 가져와야 할일이 생겼다.  
하지만 찾아보니 기존에 spring 에서 하던 방법으로는 안될 것 같다. 왜냐하면 [spring boot 에서는 jar 로 사용할 때는 jsp를 사용할 수 없다고 한다.](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-web-applications.html#boot-features-jsp-limitations)  
내용을 읽어보니 boot에 내장 tomcat에 하드코딩 패턴때문에 jar형식으로는 webapp내용을 가져올 수 없다고 한다. 그리고 공식적으로 jsp를 지원하지 않는다고 한다. boot에서 밀고 있는 template engine 들이 여러개 있었는데 간단한 view 하나 추가하는데 공수가 많이 들게되면 좋지 않을꺼라 생각해서 jsp로 view 를 구성하는 방법을 시도해보았다.  
일단 작업을 시작하기 전에 현재 사용하고 있는 버전들을 정리하고 간다.  

### 사용하고 있는 버전은 다음과 같다.
* spring boot 1.5.7
* gradle 4.4  

### 나중에는 없어질 view 이지만
front작업이 react로 완료되면 이 view 는 더이상 필요하지 않다.  
그래서 나는 최소한의 공수로 기존에 있는 jsp 파일을 사용하여 가볍게 포팅만 하고자 했다.  

--- 

## 1차 시도
spring boot 에서 jsp view를 사용하기 위해 spring에서 구성하는 방법과 추가적으로 필요한 설정들을 해주었다.  
spring boot 의 내장 tomcat에는 jsp parser가 없기 때문에 의존 패키지를 추가해주어야 한다.  

- build.gradle  

```gradle
derendencies {
    compile('javax.servlet:jstl')
    compile("org.apache.tomcat.embed:tomcat-embed-jasper")
}
``` 
그리고 구조는 아래와 같이 구성했다. main밑에 webapp폴더를 추가해서 jsp파일을 추가해준다.

```
.
├── build.gradle
├── gradlew
├── gradlew.bat
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── example
    │   │           └── demo
    │   │               ├── DemoApplication.java
    │   │               └── MyController.java
    │   ├── resources
    │   │   └── application.properties
    │   └── webapp
    │       └── WEB-INF
    │           └── jsp
    │               └── index.jsp
    └── test
```  
spring boot 는 webapp의 위치를 모르기 때문에 설정파일에 경로를 명시해주어야 한다.  

* application.properties  

```
spring.mvc.view.prefix=/WEB-INF/jsp/
spring.mvc.view.suffix=.jsp
```
설정은 다했다. 이제 controller에서 view를 호출해보자.
* MyController.java

```java
package com.example.demo;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.ModelAndView;

@Controller
public class MyController {

    @RequestMapping(value = "/")
    public ModelAndView main() {
        ModelAndView view = new ModelAndView("index");
        view.addObject("text", "world");
        return view;
    }
}
```
설정파일에서 prefix, suffix를 적어주었기 때문에 view이름은 파일이름만 넣어주면 된다.  

* index.jsp

```html
<html>
    <body>
        <h1>Hello world</h1>
        hello ${text}
    </body>
</html>
```
이제 bootRun 을 하면 build 를 하고 테스트를 해볼 수 있다.  

```bash
$ ./gradlew bootRun
:compileJava UP-TO-DATE
:processResources UP-TO-DATE
:classes UP-TO-DATE
:findMainClass
:bootRun
```

* localhost:8080/
```html
Hello world

hello world
```
잘 (~~뜨는것 처럼~~)뜨는 것을 확인할 수 있다. 여기까지만 보면 생각보다 간단히 처리할 수 있다고 생각할 수 있다.  
우리가 원하는 것은 빌드해서 jar로 만들어서 호출이 잘 되어야 하기 때문에 아직 모든걸 완성했다고 할 수 없다.  
gradlew build를 해보자  

```sh
$ ./gradlew build
:compileJava UP-TO-DATE
:processResources UP-TO-DATE
:classes UP-TO-DATE
:findMainClass
:jar
:bootRepackage
:assemble
:compileTestJava UP-TO-DATE
:processTestResources NO-SOURCE
:testClasses UP-TO-DATE
:test UP-TO-DATE
:check UP-TO-DATE
:build

BUILD SUCCESSFUL

Total time: 1.494 secs
```

`./build/libs/testGradle-0.0.1-SNAPSHOT.jar` 에 jar가 만들어졌다. 이걸로 직접 띄워서 호출해보자.  

```html
<html>
    <body>
        <h1>Whitelabel Error Page</h1>
        <p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p>
        <div id='created'>Thu Feb 22 18:37:10 KST 2018</div>
        <div>There was an unexpected error (type=Not Found, status=404).</div>
        <div>/WEB-INF/jsp/index.jsp</div>
    </body>
</html>
```

에러내용을 보면 `/WEB-INF/jsp/index.jsp` 경로에 파일을 못찾았기 때문이다.  
bootRun 명령어는 기본적으로 [processResources의 결과물을 사용하지 않는다고 한다.]("https://docs.spring.io/spring-boot/docs/current/reference/html/build-tool-plugins-gradle-plugin.html#build-tool-plugins-gradle-running-applications")  
 그렇기 때문에 개발시에 동적으로 resource를 추가할 수도 있지만 (약간의 꼼수지만) webapp 폴더에도 접근이 가능한 이유이다.  
이와는 달리 build명령어는 processResources를 통해 resources의 정적 파일들을 제외하고는 포함을 하지 않기 때문에 webapp 폴더의 내용은 jar파일에 포함되지 않았던 문제이다.  
  
여기에서 막혀서 jsp파일들을 사용할 수 없나 싶었다.  
하지만 이미 칼을 뽑았기에 무라도 썰어보자.   

## 2차 시도 
여기서 그냥 jsp파일을 쓰지 않고 다른 template engine들을 쓸까도 생각했었다.  
그런데 문득 jsp파일들을 resources폴더에 넣으면 되지않을까? 라는 생각으로 좀 더 파고 들어가보기로 했다.  
