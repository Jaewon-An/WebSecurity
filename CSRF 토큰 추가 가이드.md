# CSRF 토큰 추가 가이드

Spring Security를 이용한 CSRF 토큰 구현 가이드입니다. (Spring Security 5.8.x 기준)

**버전:** v20260127

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

## 5. 인코딩 필터에 forceEncoding 파라미터 추가

인코딩 필터에 `forceEncoding` 파라미터를 추가합니다.

> **필수:** 이 설정이 없으면 한글이 깨질 수 있습니다.

```xml
<filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>utf-8</param-value>
    </init-param>
    <init-param>
        <param-name>forceEncoding</param-name>
        <param-value>true</param-value>  <!-- 이거 필수! -->
    </init-param>
</filter>
```

---

## 6. web.xml multipart-config 설정 추가

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

## 7. JSP 파일 수정

### 7.1 태그라이브러리 선언

JSP 파일 상단에 Spring Security 태그라이브러리를 선언합니다.

```jsp
<%@ taglib prefix="sec" uri="http://www.springframework.org/security/tags" %>
```

---

## 8. JSP 메타 태그 추가

`<head>` 섹션에 CSRF 메타 태그를 추가합니다.

```jsp
<sec:csrfMetaTags />
```

---

## 9. JSP AJAX 전역 설정

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

---

## 10. JSP Form 설정

### 10.1 Form 메소드 변경

Spring Security는 GET 메소드는 토큰 체크를 하지 않으므로, Form의 메소드를 POST로 변경합니다.

```jsp
<form method="post" action="...">
```

### 10.2 CSRF 토큰 필드 추가

Form 내부에 CSRF 토큰 hidden 필드를 추가합니다.

```jsp
<form method="post" action="...">
    <sec:csrfInput/>
    <!-- 나머지 폼 필드 -->
</form>
```

---

## 구현 체크리스트

구현 시 아래 항목을 순서대로 확인하세요.

- [ ] **1단계:** pom.xml에 Spring Security 의존성 추가
- [ ] **2단계:** context-security.xml 파일 생성 및 설정
- [ ] **3단계:** dispatcher-servlet.xml의 multipartResolver 변경
- [ ] **4단계:** web.xml에 Spring Security Filter Chain 등록
- [ ] **5단계:** web.xml 인코딩 필터에 forceEncoding 파라미터 추가 ⭐
- [ ] **6단계:** web.xml의 DispatcherServlet에 multipart-config 추가
- [ ] **7단계:** JSP에 Spring Security 태그라이브러리 선언
- [ ] **8단계:** JSP에 CSRF 메타 태그 추가
- [ ] **9단계:** JSP에 AJAX 전역 설정 추가
- [ ] **10단계:** Form 메소드를 POST로 변경하고 CSRF 토큰 필드 추가

---

## 주의사항

### 필터 순서
```
1. Encoding Filter (forceEncoding=true)
   ↓
2. Spring Security Filter Chain
```

인코딩 필터가 Spring Security Filter Chain보다 먼저 등록되어야 한글이 깨지지 않습니다.

### CSRF 예외 경로
- GET 메소드는 기본적으로 CSRF 검사에서 제외됩니다.
- 특정 경로를 예외로 설정해야 하는 경우 `context-security.xml`의 `csrfRequestMatcher`에 추가하세요.

### 파일 업로드
- `StandardServletMultipartResolver` 사용 시 반드시 `<multipart-config/>`를 DispatcherServlet에 추가해야 합니다.
- 기존 `CommonsMultipartResolver`를 사용하면 CSRF + multipart/form-data 조합에서 403 오류가 발생할 수 있습니다.

---

## 버전 정보

- **Spring Security 버전:** 5.8.16
- **작성일:** 2026-01-27
- **적용 환경:** Spring Framework XML 설정 방식

---

## 문제 해결

### 한글이 깨지는 경우
1. 인코딩 필터에 `forceEncoding=true` 설정 확인
2. 인코딩 필터가 Spring Security Filter Chain보다 먼저 등록되었는지 확인

### 파일 업로드 시 403 오류
1. `StandardServletMultipartResolver` 사용 확인
2. `<multipart-config/>` 설정 확인

### AJAX 요청 시 403 오류
1. CSRF 메타 태그 추가 확인
2. AJAX 전역 설정 확인
3. 브라우저 개발자 도구에서 요청 헤더에 `X-CSRF-TOKEN`이 포함되었는지 확인
