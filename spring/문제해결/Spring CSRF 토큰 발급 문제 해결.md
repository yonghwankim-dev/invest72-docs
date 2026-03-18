
## 개요
개인 프로젝트인 invest72 프로젝트에서 React, Spring 기반으로 Google 소셜 로그인 기능을 세션 쿠키 방식으로 구현하였습니다. 로그인 기능을 기능적으로는 잘 동작하였지만 **CSRF(Cross-Site Request Forgery, 교차 사이트 요청 위조)** 설정은 구현하지 않아서 CSRF 보안 위험성을 가지고 있다는 것을 알게 되었습니다.
이 글에서는 다음 내용을 설명할 예정입니다.
- CSRF 공격이 무엇인지 개념과 공격의 작동 과정을 설명합니다. 
- CSRF 공격을 예방하기 위한 CSRF 토큰을 이용한 작동 과정을 설명합니다.
- React 및 Spring 기반으로 CSRF 공격을 예방하기 위한 구현을 설명합니다.
	- 백엔드 서버인 Spring 서버에서는 Spring Security 프레임워크를 기반으로 CSRF 보안을 위한 설정을 추가 및 구현합니다.
	- 프론트엔드 서버에서는 axios 설정 부분에 CSRF 설정을 추가하는 것을 구현합니다.
- CSRF 토큰을 적용하는 도중에 발생한 문제 해결 경험을 소개합니다.

## CSRF 공격은 무엇인가?
**CSRF(Cross-Site-Request-Forgery) 공격은 공격자가 사용자나 브라우저를 속여서 악성 사이트로부터 대상 사이트로 HTTP 요청을 보내도록 유도하는 공격**입니다. 이 요청에는 사용자의 인증 정보가 포함되어 있습니다. 대상 사이트인 서버는 HTTP 요청을 받으면 사용자가 요청한 것으로 오해하여 해로운 작업을 수행하게 됩니다. 즉, **CSRF 공격은 사용자의 인증 정보를 탈취해서 서버에 HTTP 요청해서 해로운 작업을 수행하게 하는 공격**입니다.

## CSRF 공격은 어떻게 작동하는가?
웹 사이트는 일반적으로 사용자의 브라우저로부터 HTTP 요청을 받아서 사용자를 대신해서 작업을 수행합니다. 예를 들어 상품을 구매하거나 예약하는 것과 같은 작업이 있습니다. 이때 HTTP 요청이 해당 사용자로부터 온것인지 보장하기 위해서 서버는 HTTP 요청에 사용자의 인증 정보가 포함되기를 원합니다. 대표적으로 사용자의 세션 ID가 담긴 **쿠키(Cookie)**가 대표적입니다.

예를 들어 사용자는 이전에 자신의 은행 사이트에 로그인한 상태이며, 브라우저에는 해당 사용자의 세션 쿠키가 저장되어 있습니다. 이 페이지(my-bank.example.org 브라우저 폼)에는 다른 사람에게 송금할 수 잇는 `<form>` 요소가 포함되어 있습니다. 사용자가 폼을 제출하면 브라우저는 폼 데이터(recipient, amount)를 포함한 `POST` HTTP 요청을 서버로 전송합니다. 만약 사용자가 로그인된 상태라면, 이 요청에는 사용자의 쿠키(user=1234567)가 포함됩니다. 서버는 쿠키를 검증한 뒤에, 특수한 작업(해당 예시에서는 송금 작업)이 수행될 것입니다.
![](../../images/Pasted%20image%2020260316162918.png)

이 글에서는 위 설명에 작성한 특수한 작업을 **"상태 변경 요청"**이라고 부르겠습니다.

이번에는 CSRF 공격이 어떻게 작동되는지 설명합니다. CSRF 공격에서 공격자는 `form`이 포함된 웹 사이트(free-kittens.org)를 생성합니다. `form` 요소의 `action` 속성에는 은행의 웹사이트로 설정합니다. 그리고 은행의 필드들을 흉내낸 hidden 속성을 가진 `input`  필드들을 포함합니다.
다음 코드를 보면 송금 기능을 요청하고 있고, 수신인은 공격자로 되어 있습니다. 금액은 1000으로 설정되어 있습니다.
<form action="https://my-bank.example.org/transfer" method="POST">
  <input type="hidden" name="recipient" value="attacker" />
  <input type="hidden" name="amount" value="1000" />
</form>
```html
<form action="https://my-bank.example.org/transfer" method="POST">
  <input type="hidden" name="recipient" value="attacker" />
  <input type="hidden" name="amount" value="1000" />
</form>
```

해당 페이지에는 또한 페이지 로드시 폼을 제출하는 자바스크립트 코드가 포함되어 있습니다.
```js
const form = document.querySelector("form");
form.submit();
```

사용자가 공격자의 웹페이지(free-kittens.org)를 방문하면, 브라우저는 은행의 서버(Server my-bank.example.org)에 폼을 제출합니다. 사용자는 은행 사이트에 로그인한 상태이기 때문에 요청은 사용자의 실제 쿠키가 포함됩니다. 그래서 세션 쿠키가 존재하기 때문에 은행의 서버는 성공적으로 요청을 검증하고, 현금을 해커에게 송금하게 됩니다.
![](../../images/Pasted%20image%2020260316164243.png)

공격자가 CSRF 공격하는 또 다른 방법이 있습니다. 예를 들어, 웹 사이트가 특정 작업을 수행하기 위해 `GET`  요청을 사용한다면, 공격자는 폼을 사용할 필요조차 없게 됩니다. 대신 공격자는 사용자에게 아래와 같은 마크업이 포함된 페이지 링크를 보내는 것만으로도 공격을 실행할 수 있습니다.
```html
<img src="https://my-bank.example.org/transfer?recipient=attacker&amount=1000" />
```

위 이미지 태그의 링크와 같이 작성하면, 사용자가 페이지를 로드하면, 브라우저는 이미지를 가져오려고 시도합니다. 하지만 그 리소스는 바로 "송금 요청"이 됩니다. 물론 HTTP 요청은 GET 요청으로 하게 됩니다. 그런데 서버에서 GET 요청임에도 불구하고 실제 서버 구현은 송금을 하도록 구현되어 있다면 송금 작업이 발생할 것입니다. 송금 작업과 같은 상태 변경 요청은 대개 "POST" 요청으로 설계해야 합니다. 그럼에도 불구하고 상태 변경 작업을 GET 메서드로 설계하면 위 사례와 같이 CSRF 공격을 쉽게 당할 수 있게 만듭니다.

일반적으로 CSRF 공격은 여러분들의 애플리케이션 서버가 다음과 같은 특징을 가지는 경우에 CSRF 공격이 가능해집니다.
- 서버에 몇몇 상태를 변경하는 HTTP 요청(POST, UPDATE, DELETE, PATCH)을 사용하는 경우
- 해당 요청이 인증된 사용자로부터 온것인지 확인하기 위해서 오직 "쿠키"만을 사용하는 경우
- 공격자가 예측할 수 있는 매개변수만을 요청에 사용되는 경우

## CSRF 공격 예방하기
CSRF 공격을 막기 위한 4가지 방법을 소개합니다.
- 첫번째는 클라이언트(브라우저)가 CSRF 토큰을 헤더에 같이 보내도록 하세요. 이는 위 예시에서 살펴본 `form` 요소를 통해 상태 변경 요청을 보내는 경우를 예방하기 위한 방법입니다.
- 두번째 방법은 [페치 메타데이터(Fetch metadata) HTTP 헤더](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/CSRF#fetch_metadata)를 사용하여, 해당 상태 변경이 "cross-site" 요청(sec-fetch-site 헤더의 값이 "same-origin" 또는 "same-site" 인지 검증)인지 여부를 확인합니다.
- 세번째 방법은 상태를 변경하는 요청이 "단순 요청(Simple Requests)"이 되지 않도록 보장하여, 교차 출처(Cross-Origin) 요청이 기본적으로 차단되도록 만드는 것입니다. 이 방법은 `fetch()`와 같은 자바스크립트 API를 사용하여 상태 변경을 요청을할때 적합합니다.

마지막 방법으로 위 3가지 방법들과 별개로 CSRF 공격을 방어하기 위해서 SameSite 쿠키 속성에 대해서 알아볼 것입니다.

## CSRF Token은 무엇인가?
서버는 페이지를 제공할때 **"CSRF Token"이라 불리는 예측 불가능한 값**을 해당 페이지에 삽입합니다. 이후 정상적인 페이지가 서버로 상태 변경 요청을 할때, HTTP 요청에 이 CSRF 토큰을 포함하여 전송합니다. 그러면 서버는 토큰 값을 확인하고, 값이 일치할 때만 요청을 수행합니다. 공격자는 이 CSRF 토큰 값을 추측할 수 없기 때문에 "상태 변경 요청"을 할수 없습니다. 설령 공격자가 이미 사용된 토큰을 알아내도 매번 토큰이 변경된다면, 해당 요청을 다시 제현(replay)할 수 없습니다.


> [!NOTE] CSRF 토큰을 로그인할때 한번만 쿠키로 발급하는 경우
> 예를 들어 서버로부터 CSRF 토큰을 매번 발급받아 제출하는 형태가 아닌, 현재 우리 서버는 로그인시 CSRF 토큰을 하나 발급하여 쿠키에 저장하는 방식을 취하고 있는데, 만약 해커가 CSRF 토큰 쿠키를 탈취하여 같이 전달하면 위조 공격이 통하지 않을까요?
> 현재 웹서버와 웹 애플리케이션 서버는 서로 다른 도메인을 가지고 있고, 웹서버에서 웹 애플리케이션 서버로 CSRF 토큰값을 전송하기 위해서는 자바스크립트로 토큰 쿠키 값을 읽어야 합니다. 그래서 발급받은 CSRF 토큰 쿠키의 옵션은 `HttpOnly=false, SameSite=None`가 됩니다. 이러한 옵션 설정으로 인하여 해커의 사이트또한 CSRF 토큰 쿠키의 값을 읽을수 있습니다.
> - **Same-Origin Policy에 의해서 해커가 만든 사이트에서 사용자의 브라우저를 통해 대상 서버의 쿠키를 직접 읽으려 하면, 브라우저가 이를 차단합니다.**
> - XSS(Cross-Site Scripting)의 필요성 : 해커가 웹서버의 내부 스크립트에 악성 코드를 심을 수 있어야(XSS 공격 성공) 비로소 `document.cookie`를 통해 토큰을 빼낼수 있습니다.

폼 제출의 경우, CSRF 토큰은 보통 히든 폼 필드에 포함됩니다. 이를 통해 폼이 제출될 때 토큰이 서버로 자동 전송되어 검증을 거칠수 있게 됩니다.

자바스크립트 `fetch()` 함수의 경우에 토큰은 쿠키에 담겨 있거나, 페이지 내에 삽입될 수 있습니다. 그러면 자바스크립트가 이 값을 추출하여 추가적인 HTTP 헤더로 전송합니다.

현대 웹 프레임워크는 보통 CSRF 토큰을 내장 지원합니다. 예를 들어 장고(Django)는 `csrf_token` 태그를 사용하여 폼 보호를 클라이언트에게 제공합니다. 이는 토큰이 포함된 추가적인 히든 폼 필드를 생성합니다. 그리고 나서 프레임워크는 이후 서버에서 이 토큰을 검증합니다.

이러한 보호 기능을 제대로 활용하려면, 웹 사이트 내에서 상태 변경 요청을 사용하는 모든 지점을 파악하고 있어야 하며, 선택한 프레임워크가 제공하는 방어 기법이 해당 지점들에 빠짐없이 적용되어 있는지 확인해야 합니다.

### CSRF 토큰을 포함한 동작과정
![](../../images/Pasted%20image%2020260318114938.png)
1. 클라이언트는 서버에 로그인을 요청합니다.
2. 서버는 로그인 기능을 처리하고 CSRF 토큰을 신규 발급합니다. 신규 발급된 CSRF 토큰은 서버에서 저장하고 클라이언트들에게는 Set-Cookie로 쿠키를 구어서 전달합니다.
3. CSRF 토큰 쿠키를 받은 클라이언트가 이후 POST 요청을 할때 헤더와 쿠키로 CSRF 토큰을 서버에 전달합니다.
4. 요청을 받은 서버는 우선 CSRF 토큰 값을 검증합니다.
	1. 만약 서버는 토큰 값이 유효하면 POST 요청을 수행합니다.
	2. 만약 서버가 토큰 값이 유효하지 않으면 403 응답합니다.
5. 클라이언트는 POST 요청에 대한 응답을 받습니다.

위 동작과정의 경우에는 로그인 인증을 하면서 CSRF 토큰을 신규 발급하는 경우입니다. 로그인 외에도 예를 들어 단순 GET을 서버에 요청해서 토큰만을 발급받을 수 있거나 할수 있습니다.
서버가 HTML 페이지까지 생성하는 서버사이드 렌더링 구조라면 클라이언트가 별도로 토큰을 요청하지 않고도 서버에서 미리 생성하여 전달할 수도 있습니다.

---

## React, Spring 프레임워크 기반 CSRF 토큰 설정 구현하기
저의 개인 프로젝트인 invest72 서비스의 구성은 프론트엔드는 React, 백엔드 서버는 Spring 프레임워크 기반으로 구현하였습니다. 이번 챕터에서는 invest72 프로젝트에 CSRF 공격을 예방하기 위한 CSRF 설정을 구현하는 것을 소개합니다. 그리고 구현하면서 발생한 문제를 해결한 경험을 공유합니다. 백엔드 서버 같은 경우에는 CSRF 설정을 수월하게 하기 위해서 Spring Security 프레임워크 기반으로 구현하였습니다. React 라이브러리 같은 경우에는 서버에 요청하기 위한 인스턴스로 axios 라이브러리를 사용하였습니다.

### Spring Security CSRF 설정 구현하기
제가 설계한 백엔드 서버의 CSRF 설계는 다음과 같습니다.
- 클라이언트가 비로그인 상태로 서버에 GET 요청을 포함한 어떤 요청을 하면 서버에서는 CSRF 토큰이 발급된 적이 없다면 신규 발급해서 쿠키 방식으로 응답해야 합니다.
- 클라이언트가 서버에 로그인을 하게되면 기존 CSRF 토큰을 폐기하고 신규 CSRF 토큰을 발급하여 쿠키 방식으로 응답해야 합니다.
	- 클라이언트-서버 인증 방식은 세션/쿠키 방식을 사용하고 있습니다.
- 클라이언트가 서버에 로그아웃을 요청하게 되면 CSRF 토큰을 폐기해야 합니다.
- 클라이언트가 POST, DELETE, UPDATE 메서드와 같은 상태 변경 요청을 하게되면 서버는 CSRF 토큰을 검증해야 합니다.
	- CSRF 토큰을 검증하는 부분은 이미 Spring Security의 `CsrfFilter` 필터를 제공하고 있습니다.
	- 토큰이 유효하지 않다면 403 응답합니다.
- 서버는 특정 출처(origin)만을 명시하여 특정 출처에서만 요청을 받도록 CORS(Cross-Origin-Resource-Sharing) 설정을 해야 합니다.
- 웹서버 및 웹 애플리케이션 서버의 도메인은 다릅니다.
	- 웹서버(React) : `invest72.web.app`
	- 웹 애플리케이션 서버(Spring) : `invest72-api.duckdns.org`

### Spring Security FilterChain CSRF 설정
SecurityFilterChain 설정
- `CsrfTokenRequestAttributeHandler` : HttpServletRequest 객체의 Attribute에 csrfToken 객체 및 이름을 설정하는 핸들러입니다. 해당 핸들러가 있어야 다른 Request 객체를 사용하는 곳에서도 Request 객체를 이용하여 CSRF 토큰을 참조할 수 있습니다.
	- 해당 핸들러는 `csrfFilter` 필터에서 호출합니다.
```java
http  
    .csrf(configurer ->  
       // 쿠키 기반의 CSRF 토큰 저장소 설정 (프론트엔드가 토큰을 읽을 수 있게 함)  
       // withHttpOnlyFalse()로 설정하여 JavaScript에서 CSRF 토큰에 접근할 수 있도록 허용  
       configurer.csrfTokenRepository(cookieCsrfTokenRepository())  
          // 요청 헤더명 지정 (기본값은 X-XSRF-TOKEN)
			          .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())  
    )
```

**cookieCsrfTokenRepository 설정 구현**
- `withHttpOnlyFalse` : 생성되는 쿠키의 옵션 `HttpOnly=false`로 설정됩니다. 이는 클라이언트에서 쿠키 값을 읽어서 `X-XSRF-TOKEN` 헤더에 전달하기 위함
- `secure=true` : 클라이언트와 서버간에 HTTPS 프로토콜로만 전송 및 수신하도록 설정
- `httpOnly=false` : 자바스크립트에서 쿠키 값을 읽을수 있도록 설정
- `sameSite=None` : 서버가 출처(origin)가 다른 클라이언트에게 쿠키를 전달하도록 None으로 설정
	- Strict : 브라우저와 서버가 서로 다른 출처인 경우 요청에 쿠키를 포함하여 전송하지 않습니다.
	- Lax : 브라우저와 서버가 서로 다른 출처여도 다음과 같은 요청인 경우에는 쿠키를 포함하여 전송합니다.
		- 요청이 최상위 브라우징 컨텍스트의 이동
		- GET 요청인 경우
		- 단, `<iframe>` 안에서의 페이지 이동의 경우에는 쿠키가 전송되지 않음.
	- None : 브라우저와 서버가 서로 다른 출처(origin)여도 쿠키를 보냅니다.
```java
private CookieCsrfTokenRepository cookieCsrfTokenRepository() {  
    CookieCsrfTokenRepository repository = CookieCsrfTokenRepository.withHttpOnlyFalse();  
    repository.setCookieCustomizer(cookie -> {  
       cookie.path("/"); // 쿠키 경로 설정  
       cookie.secure(true); // HTTPS에서만 전송  
       cookie.httpOnly(false); // JavaScript에서 접근 가능하도록 설정  
       cookie.sameSite("None"); // SameSite=None으로 설정하여 크로스사이트 요청에서도 쿠키가 전송되도록 함  
    });  
    return repository;  
}
```

**CORS 설정**
SecurityFilterChain 스프링 빈 생성시 다음과 같이 설정합니다.
```java
@Bean  
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
    // ...
    http  
       .cors(cors ->cors.configurationSource(corsConfigurationSource()))
    // ...
}

@Bean  
public CorsConfigurationSource corsConfigurationSource() {  
    CorsConfiguration configuration = new CorsConfiguration();  
    configuration.setAllowedOrigins(corsConfigurationProperties.getAllowedOrigins());  
    configuration.setAllowedMethods(corsConfigurationProperties.getAllowedMethods());  
    configuration.setAllowedHeaders(corsConfigurationProperties.getAllowedHeaders());  
    configuration.setAllowCredentials(corsConfigurationProperties.getAllowCredentials());  
  
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();  
    source.registerCorsConfiguration("/**", configuration);  
    return source;  
}
```

corsConfigurationProperties가 참조하는 CORS 설정은 다음과 같습니다.
- allowed-headers 프로퍼티 리스트 부분에서 "X-XSRF-TOKEN" 헤더를 추가해야지 서버가 해당 헤더를 받을 수 있습니다.
```yaml
app:  
  cors:  
    allowed-origins: [ "https://invest72.web.app", "https://invest72.firebaseapp.com" ]  
    allowed-methods: [ "GET", "POST", "PUT", "PATCH","DELETE", "OPTIONS" ]  
    allowed-headers: [ "Authorization", "Content-Type", "Cache-Control", "X-XSRF-TOKEN" ]  
    allow-credentials: true
```

**csrfCookieFilter 필터 구현**
- 해당 필터는 CsrfToken이 발급되지 않았다면 신규 발급하고 쿠키를 구어서 클라이언트에게 응답하는 필터입니다.
- 사용자가 비로그인 상태에서도 토큰을 발급하여 POST 요청에 대해서 검증하기 위함입니다.
- 하지만 해당 필터는 `BasicAuthenticationFilter` 필터 이후에 작동하기 때문에 로그인 후 리다이렉션 하는 경우 작동하지 않습니다. 따라서 로그인 처리후 리다이렉션 하는 과정에서 클라이언트에게 CSRF 토큰 쿠키(XSRF-TOKEN)을 전달하기 위해서는 별도의 로직 추가가 필요합니다. 
```java
@Slf4j
public class CsrfCookieFilter extends OncePerRequestFilter {
	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
		FilterChain filterChain) throws ServletException, IOException {
		CsrfToken csrfToken = (CsrfToken)request.getAttribute(CsrfToken.class.getName());
		if (csrfToken != null) {
			// 토큰을 명시적으로 get하는 시점에 CookieCsrfTokenRepository가 쿠키를 생성 및 저장합니다.
			csrfToken.getToken();
			log.debug("CSRF Token added to response cookies: {}", csrfToken.getToken());
		}
		filterChain.doFilter(request, response);
	}
}
```

**CsrfCookieFilter 등록**
SecurityFilterChain 스프링 빈 생성시 다음과 같이 등록합니다.
```java
@Bean  
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
    // 필터 등록  
    http.addFilterAfter(new CsrfCookieFilter(), BasicAuthenticationFilter.class); // 인증 필터 이후에 실행
    // ...
}
```

**로그인 성공후 CSRF 토큰을 쿠키로 전달 구현**
- 현재 백엔드 서버의 로그인 같은 경우에는 소셜 로그인 기능입니다.
- 소셜 로그인에 성공하면 클라이언트에게 리다이렉션 응답을 하기 때문에 인증 필터 이후에 `CsrfCookieFilter` 필터는 작동하지 않습니다.
- 따라서 로그인 이후에 새로운 CSRF 토큰 쿠키를 구어서 응답하기 위해서는 SuccessHandler에 별도의 로직이 구현되어야 합니다.
```java
@Override  
public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,  
    Authentication authentication) throws IOException, ServletException {  
    // 로그인 성공 시점의 새로운 CSRF 토큰을 강제로 생성 및 로드  
    CsrfToken csrfToken = (CsrfToken)request.getAttribute(CsrfToken.class.getName());  
    if (csrfToken != null) {  
       // 이 호출이 CookieCsrfTokenRepository를 트리거하여 새 Set-Cookie 헤더를 만듭니다.  
       csrfToken.getToken();  
       // csrfToken 값 로그 출력 (디버깅용)  
       log.debug("New CSRF Token generated on authentication success: {}", csrfToken.getToken());  
    }  
  
    // redirectUri는 프론트에서 로그인 성공 후 리다이렉트할 URL입니다. 예를 들어, "http://localhost:3000/login-success"와 같은 URL이 될 수 있습니다.  
    String targetUrl = UriComponentsBuilder.fromUriString(redirectUri).build().toUriString();  
    getRedirectStrategy().sendRedirect(request, response, targetUrl);  
}
```

왜 CsrfCookieFilter를 인증 필터(BasicAuthenticationFilter) 이후에 배치하나요?
- CsrfCookieFilter를 인증 필터 이전에 배치하면 해당 필터가 CSRF 토큰을 발급하고 쿠키를 생성하지 않을까요?
- Spring Security 내부 전략에 의해서 기존 CSRF 토큰이 폐기되고, 새 토큰이 생성됩니다.
	- AbstractAuthenticationProcessingFilter에서 인증 처리후에 SessionStrategy에 의해서 기존 CSRF 토큰이 폐지됩니다.
- 인증 필터 이후에 배치된 CsrfCookieFilter는 새 토큰을 가로채서 브라우저에 쿠키로 전달됩니다.
- 만약 인증 필터 이전에 두게 되면 로그인이 성공해서 토큰이 변경되더라도 그 직전 상태의(이미 폐기된) 토큰을 쿠키로 굽게되는 실수가 발생할 수 있습니다.

**로그아웃시 CSRF 토큰 폐기는 어떻게 작동하는가?**
- Spring Security에서 제공하는 CsrfLogoutHandler가 등록되어 폐기를 진행합니다.
- 해당 핸들러는 csrf 설정을 등록하면 기본적으로 등록됩니다.
```java
public final class CsrfLogoutHandler implements LogoutHandler {  
  
    private final CsrfTokenRepository csrfTokenRepository;  
  
    /**  
     * Creates a new instance     * @param csrfTokenRepository the {@link CsrfTokenRepository} to use  
     */    public CsrfLogoutHandler(CsrfTokenRepository csrfTokenRepository) {  
       Assert.notNull(csrfTokenRepository, "csrfTokenRepository cannot be null");  
       this.csrfTokenRepository = csrfTokenRepository;  
    }  
  
    /**  
     * Clears the {@link CsrfToken}  
     *     * @see org.springframework.security.web.authentication.logout.LogoutHandler#logout(jakarta.servlet.http.HttpServletRequest,  
     * jakarta.servlet.http.HttpServletResponse,     * org.springframework.security.core.Authentication)     */    @Override  
    public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {  
       this.csrfTokenRepository.saveToken(null, request, response);  
    }  
  
}
```


### React axios CSRF 설정 추가
React 코드에서 axios 인스턴스 생성시 다음과 같이 설정합니다.
- `withXSRFToken` : 해당 설정을 true로 설정해야지 axios가 HTTP 요청시 X-XSRF-TOKEN 헤더에 XSRF-TOKEN 쿠키의 값을 자동으로 읽어서 헤더에 전달합니다.
- `withCredentials` : true로 설정하면 HTTP 요청에 쿠키를 포함하게 됩니다.
```js
import axios from "axios";

const HOST = process.env.REACT_APP_API_BASE_URL || "http://localhost:8080";
const api = axios.create({
        baseURL: HOST,
        withCredentials: true, // 모든 세션 요청에 쿠키 포함 
        xsrfCookieName: "XSRF-TOKEN", // CSRF 토큰 쿠키 이름
        xsrfHeaderName: "X-XSRF-TOKEN", // CSRF 토큰 헤더 이름
        withXSRFToken: true, // CSRF 토큰 자동 포함
    });
export default api;
```

### 실행 결과 확인
클라이언트에서 서버로 소셜 로그인을 요청합니다. 소셜 로그인 실행 결과를 보면 리다이렉트를 응답하면서 Set-Cookie에 INVEST72_SESSION 쿠키와 XSRF-TOKEN 쿠키를 전달하고 있는 것을 볼수 있습니다.
XSRF-TOKEN 쿠키는 2개로 응답하는데 값이 공백인 것은 기존 XSRF-TOKEN 쿠키 값이 존재하는 경우 기존 토큰 쿠키값을 제거하기 위해서 응답하는 것입니다. 그래서 아래것은 신규 토큰 쿠키의 것입니다.
기존 XSRF-TOKEN 쿠키를 발급받은 이유는 소셜 로그인 요청 이전에 사용자 프로필 요청(GET)을 하여 토큰을 발급받았기 때문입니다.
![](../../images/Pasted%20image%2020260318133106.png)

쿠키 저장 결과를 보면 다음과 같습니다.
- XSRF-TOKEN 같은 경우에는 이후 POST 요청 때문에 `HttpOnly=false`로 설정됨
- 클라이언트와 서버간에 도메인이 다르기 때문에 `SameSite=None`으로 설정됨
![](../../images/Pasted%20image%2020260318133402.png)

클라이언트에서 서버로 POST 요청하여 X-XSRF-TOKEN 헤더에 토큰값이 전달되는지 확인해보겠습니다.
요청 헤더를 보면 X-XSRF-TOKEN 헤더에 값이 포함되어 전달되는 것을 볼수 있습니다.
![](../../images/Pasted%20image%2020260318133602.png)


## CSRF 설정 구현 중 발생한 문제 해결
### 배경
- 프론트엔드는 React, 백엔드는 Spring인 상태에서 소셜 로그인을 세션 쿠키 방식으로 구현하였습니다. 배포 환경에서는 다른 도메인(프론트엔드는 `https://invest72.web.app`, 백엔드는 `https://invest72-api.duckdns.org`)으로 운용중입니다.
- 세션키로 발급되는 쿠키의 SameSite=None으로 설정되어 보안 위험이 존재하고, 백엔드 서버 또한 현재 CSRF를 비활성화한 상태입니다.
- 따라서 이 문제를 해결하기 위해서 Spring에 CSRF 설정을 추가하고 CSRF 토큰 기반으로 검증하고자 합니다.
- Spring 서버가 소셜 로그인을 하는 클라이언트(브라우저)에게 CSRF 토큰을 발급하여 쿠키로 전달하고자 합니다.
- 하지만 클라이언트가 로그인을 하고 받은 쿠키에는 XSRF-TOKEN 쿠키가 설정되지 않고 있습니다.
- 서버에서 CSRF 토큰을 브라우저에게 전달하고 있지 않은 상태입니다.
![](../../images/Pasted%20image%2020260316122540.png)

클라이언트에서 서버로 POST 요청을 수행한 다음에 발급받은 XSRF-TOKEN 쿠키의 현재 설정은 다음과 같습니다.
- 배포 환경에서 요구되는 XSRF-TOKEN 쿠키 설정은 HttpOnly=false, Secure=true, SameSite=None입니다. 현재 일치해야 하는 설정은 HttpOnly=false 뿐입니다.
![](../../images/Pasted%20image%2020260316123413.png)

### 원인
- 소셜 로그인(GET) 수행하고 난 이후에 클라이언트에게 세션 쿠키는 전달하지만, XSRF-TOKEN을 발급하지 않습니다. 
- 현재 Spring Security의 CsrfFilter는 CSRF 토큰을 사용하는 시점 전까지는 쿠키를 굽지 않는 지연 로딩(Lazy Loading) 방식을 사용합니다.
- 소셜 로그인이 완료되어 `successHandler`가 실행되고, 리다이렉트가 일어나는 과정에서, **서버가 클라이언트에게 명시적으로 CSRF 토큰을 조회하거나 사용하라는 명령**을 내리지 않으면 `CookieCsrfTokenRepository`는 쿠키를 생성하지 않습니다.
- POST 요청으로 XSRF-TOKEN 쿠키를 발급해도 Secure=true, SameSite=None으로 설정하지 않아서 배포 환경 상에서 전달하지 않을 것입니다.
- 현재 Spring 서버의 CORS 설정중 X-XSRF-TOKEN 헤더를 수용하지 않기 때문에 클라이언트가 CSRF 토큰을 X-XSRF-TOKEN 헤더로 전달하여도 거부합니다.

### 해결 방법
- 로그인 직후나 모든 요청에서 CSRF 토큰이 담긴 쿠키를 생성하도록 하려면, `CsrfToken`을 매번 조회하여 쿠키에 반영해주는 필터를 등록합니다.
	- `CsrfTokenRequestAttributeHandler` 설정
		- `CsrfToken`을 Request 속성으로 설정하는 핸들러
		- 필터에서 Request 객체를 이용하여 `CsrfToken` 속성을 읽기 위한 용도
	- `CsrfToken`을 조회하는 필터 추가
- XSRF-TOKEN 쿠키의 쿠키 옵션을 세밀하게 설정하기 위해서 `CookieCsrfTokenRepository` 직접 설정

**CsrfCookieFilter 구현**
```java
public class CsrfCookieFilter extends OncePerRequestFilter {
	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
		FilterChain filterChain) throws ServletException, IOException {
		CsrfToken csrfToken = (CsrfToken)request.getAttribute(CsrfToken.class.getName());
		if (csrfToken != null) {
			// 토큰을 명시적으로 get하는 시점에 CookieCsrfTokenRepository가 쿠키를 생성합니다.
			csrfToken.getToken();
		}
		filterChain.doFilter(request, response);
	}
}
```

SecurityConfig에 CsrfCookieFilter 등록
```java
http.addFilterAfter(new CsrfCookieFilter(), BasicAuthenticationFilter.class); // 인증 필터 이후에 실행
```

CookieCsrfTokenRepository 직접 설정
```java
private CookieCsrfTokenRepository cookieCsrfTokenRepository() {  
    CookieCsrfTokenRepository repository = CookieCsrfTokenRepository.withHttpOnlyFalse();  
    repository.setCookieCustomizer(cookie -> {  
       cookie.path("/"); // 쿠키 경로 설정  
       cookie.secure(true); // HTTPS에서만 전송  
       cookie.httpOnly(false); // JavaScript에서 접근 가능하도록 설정  
       cookie.sameSite("None"); // SameSite=None으로 설정하여 크로스사이트 요청에서도 쿠키가 전송되도록 함  
    });  
    return repository;  
}
```

**Spring CORS 설정**
- Spring CORS 설정시 allowed-headers 프로퍼티 리스트에 "X-XSRF-TOKEN"을 추가합니다.
- allow-credentials=true로 설정하여 다른 도메인에서 쿠기를 받기 위해서 true로 설정해야 합니다.
```yaml
app:  
  cors:  
    allowed-origins: [ "http://localhost:3000" ]  
    allowed-methods: [ "GET", "POST", "PUT", "PATCH","DELETE", "OPTIONS" ]  
    allowed-headers: [ "Authorization", "Content-Type", "Cache-Control", "X-XSRF-TOKEN" ]  
    allow-credentials: true
```

**React 클라이언트 코드 설정**
클라이언트 코드에서 axios로 요청을 보낼때 X-XSRF-TOKEN에 XSRF-TOKEN 쿠키의 값을 읽어서 헤더에 실어서 전송해야 합니다. axios 인스턴스 생성시 다음과 같이 설정합니다.
```js
import axios from "axios";

const HOST = process.env.REACT_APP_API_BASE_URL || "http://localhost:8080";
const api = axios.create({
        baseURL: HOST,
        withCredentials: true, // 모든 세션 요청에 쿠키 포함 
        xsrfCookieName: "XSRF-TOKEN", // 서버에서 발급하는 CSRF 토큰 쿠키 이름
        xsrfHeaderName: "X-XSRF-TOKEN", // 요청 헤더에 CSRF 토큰을 첨부할 때 사용할 헤더 이름
        withXSRFToken: true // CSRF 토큰 자동 첨부 활성화
    });
export default api;

```
- withXSRFToken의 기본값은 `undefined`이기 때문에 CSRF 토큰을 자동적으로 헤더에 첨부하기 위해서는 `true`로 설정해야 합니다.

## 실행 결과
금융 상품 생성 요청(POST)을 실행한 결과입니다. 요청 헤더를 보면 정상적으로 X-Xsrf-Token 헤더에 XSRF-TOKEN 쿠키의 값을 넣어서 전달하는 것을 볼수 있습니다.
![](../../images/Pasted%20image%2020260316144159.png)

## References
- https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/CSRF#overview
- https://yangbongsoo.tistory.com/5
- https://pratikchimes.medium.com/guide-to-cross-site-request-forgery-csrf-eed498f8bea3