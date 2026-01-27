# CSRF 토큰 추가 가이드

Spring Security를 이용한 CSRF 토큰 구현 가이드입니다. (Spring Security 5.8.x 기준)

---

## 1. pom.xml 의존성 추가

Spring Security 관련 의존성을 추가합니다.

```xml
<properties>
    ...
    <spring.security.version>5.8.16</spring.security.version>
    ...
</properties>

<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-web</artifactId>
    <version>${spring.security.version}</version>
</dependency>

<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-config</artifactId>
    <version>${spring.security.version}</version>
</dependency>

<!-- Spring Security JSP 태그 라이브러리 -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-taglibs</artifactId>
    <version>${spring.security.version}</version>
</dependency>
```

---

## 2. context-security.xml 파일 생성

Spring Security 설정 파일을 생성합니다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:security="http://www.springframework.org/schema/security"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/security
           http://www.springframework.org/schema/security/spring-security.xsd">

    <bean id="csrfTokenRepository"
          class="org.springframework.security.web.csrf.HttpSessionCsrfTokenRepository" />

    <!-- 인증 진입점(필수): 인증을 강제하지 않더라도 http 설정에 필요 -->
    <bean id="authenticationEntryPoint"
          class="org.springframework.security.web.authentication.Http403ForbiddenEntryPoint" />

    <security:http use-expressions="true" entry-point-ref="authenticationEntryPoint">
        <security:csrf token-repository-ref="csrfTokenRepository"
                       request-matcher-ref="csrfRequestMatcher" />
        <security:headers>
            <security:frame-options policy="SAMEORIGIN" />
        </security:headers>
        <security:intercept-url pattern="/**" access="permitAll" />
    </security:http>

    <!-- 인증을 강제하지 않더라도 SecurityFilterChain 구성을 위해 선언 -->
    <security:authentication-manager />

    <!-- CSRF 예외 처리 RequestMatcher: 안전 메서드는 제외, 특정 경로만 예외 -->
    <bean id="csrfRequestMatcher"
          class="org.springframework.security.web.util.matcher.AndRequestMatcher">
        <constructor-arg>
            <list>
                <!-- GET 메소드는 CSRF 검사 제외 -->
                <bean class="org.springframework.security.web.util.matcher.NegatedRequestMatcher">
                    <constructor-arg>
                        <bean class="org.springframework.security.web.util.matcher.OrRequestMatcher">
                            <constructor-arg>
                                <list>
                                    <bean class="org.springframework.security.web.util.matcher.AntPathRequestMatcher">
                                        <constructor-arg value="/**" />
                                        <constructor-arg value="GET" />
                                    </bean>
                                </list>
                            </constructor-arg>
                        </bean>
                    </constructor-arg>
                </bean>

                <!-- 예외 경로 1 -->
                <bean class="org.springframework.security.web.util.matcher.NegatedRequestMatcher">
                    <constructor-arg>
                        <bean class="org.springframework.security.web.util.matcher.AntPathRequestMatcher">
                            <constructor-arg value="/cmmn/juso/jusoPop.do" />
                        </bean>
                    </constructor-arg>
                </bean>

                <!-- 예외 경로 2 (필요시 추가) -->
                <bean class="org.springframework.security.web.util.matcher.NegatedRequestMatcher">
                    <constructor-arg>
                        <bean class="org.springframework.security.web.util.matcher.AntPathRequestMatcher">
                            <constructor-arg value="예외주소2" />
                        </bean>
                    </constructor-arg>
                </bean>
            </list>
        </constructor-arg>
    </bean>
</beans>
```

---

## 3. dispatcher-servlet.xml 수정

`CommonsMultipartResolver`를 `StandardServletMultipartResolver`로 변경합니다.

> **중요:** Servlet 3 기반 멀티파트 처리로 전환하여 CSRF + multipart/form-data 403 오류를 방지합니다.
> web.xml의 DispatcherServlet에 `<multipart-config/>` 추가가 필요합니다.

```xml
<!-- 파일 업로드 설정 -->
<bean id="multipartResolver"
      class="org.springframework.web.multipart.support.StandardServletMultipartResolver"/>
```

---

## 4. web.xml 필터 등록

Spring Security Filter Chain을 등록합니다.

> **주의:** 인코딩 필터 하위에 설정해야 한글이 깨지지 않습니다.

```xml
<!-- Spring Security Filter Chain (Encoding Filter 아래쪽에 추가) -->
<filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>

<filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

---

## 5. web.xml multipart-config 설정 추가

DispatcherServlet에 `<multipart-config/>` 설정을 추가합니다.

```xml
<servlet>
    <servlet-name>action</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:/springmvc/dispatcher-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
    <!-- Servlet 3 멀티파트 처리 활성화 (StandardServletMultipartResolver 사용) -->
    <multipart-config />
</servlet>
```

---

## 6. JSP 파일 수정

### 6.1 태그라이브러리 선언

JSP 파일 상단에 Spring Security 태그라이브러리를 선언합니다.

```jsp
<%@ taglib prefix="sec" uri="http://www.springframework.org/security/tags" %>
```

### 6.2 메타 태그 추가

`<head>` 섹션에 CSRF 메타 태그를 추가합니다.

```jsp
<sec:csrfMetaTags />
```

### 6.3 AJAX 전역 설정

AJAX 요청에 CSRF 토큰을 자동으로 포함하도록 설정합니다.

```jsp
<!-- AJAX 전역 설정 -->
<script>
$(document).ready(function() {
    var token = $("meta[name='_csrf']").attr("content");
    var header = $("meta[name='_csrf_header']").attr("content");

    $(document).ajaxSend(function(e, xhr, options) {
        xhr.setRequestHeader(header, token);
    });
});
</script>
```

### 6.4 Form 설정

#### Form 메소드 변경
Spring Security는 GET 메소드는 토큰 체크를 하지 않으므로, Form의 메소드를 POST로 변경합니다.

```jsp
<form method="post" action="...">
```

#### CSRF 토큰 필드 추가
Form 내부에 CSRF 토큰 hidden 필드를 추가합니다.

```jsp
<form method="post" action="...">
    <sec:csrfInput/>
    <!-- 나머지 폼 필드 -->
</form>
```

---

## 체크리스트

구현 시 아래 항목을 확인하세요.

- [ ] pom.xml에 Spring Security 의존성 추가
- [ ] context-security.xml 파일 생성 및 설정
- [ ] dispatcher-servlet.xml의 multipartResolver 변경
- [ ] web.xml에 Spring Security Filter Chain 등록
- [ ] web.xml의 DispatcherServlet에 multipart-config 추가
- [ ] JSP에 Spring Security 태그라이브러리 선언
- [ ] JSP에 CSRF 메타 태그 추가
- [ ] JSP에 AJAX 전역 설정 추가
- [ ] Form 메소드를 POST로 변경
- [ ] Form에 CSRF 토큰 필드 추가

---

## 참고사항

- Spring Security 5.8.16 버전 기준으로 작성되었습니다.
- CSRF 예외 경로는 프로젝트 요구사항에 맞게 추가/수정하세요.
- 인코딩 필터는 Spring Security Filter Chain보다 먼저 등록되어야 합니다.
