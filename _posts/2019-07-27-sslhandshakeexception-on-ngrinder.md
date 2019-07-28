---
layout: post
title: nGrinder SSLHandshakeException 에러 해결하기
date:   2019-07-27 16:30:00
author: GarlicDipping
tags:
- Programming, nGrinder
categories: Programming
---

# 서론

로드 테스트 용도로 nGrinder를 유용하게 쓰고 있던 중, 얼마 전 AWS의 Cloudfront를 붙이면서 SSLHandshakeException 문제가 발생했다.

Cloudfront에서 https 옵션을 사용할 경우 기본적으로 TLSv1.2를 지원하는데 ngrinder의 HTTPClient 코드는 이를 지원하지 않아 발생하는 것으로 보인다.

nGrinder 공식 문서나 각종 예제에서 이용하는 HTTPRequest (`net.grinder.plugin.http.HTTPRequest`) 클래스 대신 아파치 HttpClient를 래핑한 groovy 플러그인인 [http-builder](https://mvnrepository.com/artifact/org.codehaus.groovy.modules.http-builder/http-builder)를 이용하면 급한 불은 끌 수 있다. 보다 근본적으로는 nGrinder의 HTTPRequest 클래스 코드를 고치는 것이 최선이겠지만, 아파치의 HTTPBuilder 관련 클래스를 이참에 사용해 보는 것도 나쁘지 않을 것이다.

<!--more-->

## Maven Project 설정하기

http-builder 플러그인을 이용하기 위해서는 nGrinder에서 몇 가지 설정을 미리 해 두어야 한다.

1. nGrinder 컨트롤러 웹페이지에서 Groovy Maven Project를 생성
2. 이클립스에서 File-Import-Maven/Check out Maven Projects from SCM-URL 입력 후 체크아웃 
3. 자동생성되어 있는 TestRunner.groovy 한번 돌리면 안내하는 에러 메세지를 확인 
    - javaagent argument를 입력하라는 메세지를 확인했다면 OK
4. JUnit Test의 Run Configuration의 Arguments 탭 VM arguments에 -javaagent 옵션 입력

> 위 작업은 [개발자 윤준호님의 블로그 링크](https://junoyoon.tistory.com/entry/%EC%9D%B4%ED%81%B4%EB%A6%BD%EC%8A%A4%EC%97%90-Groovy-%EB%A9%94%EC%9D%B4%EB%B8%90-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EC%9E%84%ED%8F%AC%ED%8A%B8)에서 더 자세히 확인할 수 있습니다.

여기까지 설정했다면 이제 필요한 외부 디펜던시를 추가하자.

pom.xml에서 \<dependencies> 노드 안에 http-builder 및 groovy-backports-compat23 라이브러리를 추가하자.

~~~xml

<dependencies>
...
    <dependency>
        <groupId>org.codehaus.groovy.modules.http-builder</groupId>
        <artifactId>http-builder</artifactId>
        <version>0.7</version>
    </dependency>
    <dependency>
        <groupId>org.codehaus.groovy</groupId>
        <artifactId>groovy-backports-compat23</artifactId>
        <version>2.4.5</version>
    </dependency>
...
</dependencies>
~~~

> backports-compat23은 nGrinder가 사용하는 그루비 2.2.1 버전에서 ShortTypeHandling 클래스가 없어서 추가해주어야 한다.

이제 이클립스 프로젝트 뷰에서 원하는 프로젝트를 우클릭해 Maven/Update Project를 수행해 주자. pom.xml파일 변경사항이 프로젝트에 반영될 것이다. Maven Dependencies 리스트 안에 groovy-backports-compat23-2.4.5.jar과 http-builder-0.7.jar 등이 포함되었으면 성공이다. (apache httpclient 관련 라이브러리 기반이므로 해당 라이브러리 jar파일들도 추가되어 있을 것이다.)

라이브러리 추가를 마쳤다면 테스트 스크립트 파일에서 `import groovyx.net.http.HTTPBuilder` 를 입력해 보자. 오류 없이 import가 이루어진다면 작업 준비가 완료되었다.

## 예제 스크립트

[Groovy HTTPBuilder 클래스 문서](http://javadox.com/org.codehaus.groovy.modules.http-builder/http-builder/0.6/groovyx/net/http/HTTPBuilder.html)를 참고하여 Get 및 Post를 구현해 보자. grinder의 HTTPRequest와는 header, url 등등을 설정하는 방식이 다르므로 익숙해지는데 약간 시간이 걸릴 수 있다.

기본적으로는 HTTPBuilder 클래스에 base_url을 전달하고 Get 및 Post 요청을 보낼때 path 파라미터를 넘기는 방식이다. 간단히 예제를 적어둔다.

### Get

https://garlicdipping.github.io/about 페이지에 Get 요청을 보내고자 한다면 base_url은 https://garlicdipping.github.io/ 이며, path는 /about 이다.

코드로는 다음과 같이 구현할 수 있다.

~~~groovy

HTTPBuilder http = new HTTPBuilder("https://garlicdipping.github.io/");
http.get([path: "/about"]) { resp, reader ->
    int statusCode = resp.statusLine.statusCode
    String content = reader.text()
    //Implementation!
    //...
}

~~~

### Post

https://garlicdipping.github.io/posttest 라는 페이지에 파라미터를 넣어 Post 요청을 보내고자 한다면 Post Form을 Map 자료구조에 넣어 전달해야 한다. 이 예제에서는 form data 필드에 
- name: Minsoo Kim
- data: This is a test!

라는 데이터를 전달한다고 가정하자. 구현은 다음과 같다.

~~~groovy

import groovyx.net.http.HTTPBuilder
//URL Encoding 옵션을 위해 import 필요!
import static groovyx.net.http.ContentType.URLENC

def formDatas = ['name' : 'Minsoo Kim', 'data' : 'This is a test!']
HTTPBuilder http = new HTTPBuilder("https://garlicdipping.github.io/");
http.post([path: '/posttest', body: formDatas, requestContentType: URLENC]) { resp, reader ->
    int statusCode = resp.statusLine.statusCode
    String content = reader.text()
    //Implementation!
    //...
}

~~~

### Wrapping

HTTPBuilder를 통한 Get 또는 Post 요청에 대한 로직을 래핑해 statusCode와 content string을 묶은 데이터 클래스를 리턴하도록 만들어 이용했다. 코드가 짧은 편이므로  이곳에 올려둔다.

~~~groovy

package com.garlic.utils
//Status Code와 body를 래핑한 데이터 클래스
class GarlicHTTPResponse {
    public int statusCode
    public String body

    public GarlicHTTPResponse(int statusCode, String body) {
        this.statusCode = statusCode
        this.body = body
    }

    public GarlicHTTPResponse() {
        statusCode = -1
        body = null
    }
}

~~~

~~~groovy

import com.garlic.utils.GarlicHTTPResponse
import groovyx.net.http.HTTPBuilder
import groovy.util.slurpersupport.*
import static groovyx.net.http.ContentType.URLENC

//HTTPBuilder Get, Post로직 관련 래핑 클래스
class GarlicHTTP {
    private String base_url
    private HTTPBuilder http

    public GarlicHTTP(String base_url, Map headers) {
        http = new HTTPBuilder(base_url)
        http.setHeaders(headers)
    }

    public GarlicHTTPResponse get(String path) {
        GarlicHTTPResponse result = null;
        http.get([path: path]) { resp, reader ->
            int statusCode = resp.statusLine.statusCode
            String content = reader.text()
            result = new GarlicHTTPResponse(statusCode, content)
        }
        return result;
    }

    public GarlicHTTPResponse post(String path, Map params) {
        GarlicHTTPResponse result = null;
        http.post([path: path, body: params, requestContentType: URLENC]) { resp, reader ->
            int statusCode = resp.statusLine.statusCode
            def content = reader.text()
            result = new GarlicHTTPResponse(statusCode, content)
        }
        return result;
    }
}

~~~

다음과 같이 이용하면 된다.

~~~groovy

@Test
public void test(){
    //예제 용도로 적어둔 헤더
    def headers = ['Content-Type': 'text/html; charset=UTF-8',
        'Connection': 'Keep-Alive']
    GarlicHTTP http = new GarlicHTTP("https://garlicdipping.github.io/", headers)
    GarlicHTTPResponse response = http.get('/about')
}

~~~

