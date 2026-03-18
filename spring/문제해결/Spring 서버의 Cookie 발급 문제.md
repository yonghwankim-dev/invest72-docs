
## 배경
- Spring Security 프레임워크 기반으로 OAuth2 구글 소셜 로그인을 구현하였습니다.
- Spring 서버는 인증 처리를 한 다음에 클라이언트에게 쿠키에 세션 키를 전달합니다.
- 하지만 배포 환경에서 클라이언트가 서버에 로그인 요청을 하게 되면 정상적으로 쿠키에 세션키를 받을 수 있었지만, 인증이 요구되는 다른 API 요청이 실패하는 상황이 발생하였습니다.

## 원인
- 소셜 로그인을 처리후에 클라이언트가 쿠키를 받았을때 설정을 보면 옵션이 `Secure`, `HttpOnly`만 설정되어 있습니다.
- 클라이언트가 배포된 서버 주소는 `https://invest72.web.app` 이고, 서버의 주소는 `https://invest72-api.duckdns.org`입니다.
- 쿠키의 옵션 중에서 SameSite 옵션의 값이 기본값인 Lax(공백)이기 때문에 클라이언트와 서버간에 도메인이 서로 다른 크로스 사이트 환경이어서 `SameSite=Lax` 상태에서는 API 요청시 쿠키가 서버로 전송되지 않아 다른 API 요청이 거부됩니다.
![](../../images/Pasted%20image%2020260306122234.png)
![](../../images/Pasted%20image%2020260306122208.png)

## 해결 방법 및 성과
- spring 서버의 설정에서 쿠키의 SameSite 옵션을 설정합니다.
- server.servlet.session.cookie 프로퍼티 설정에서 `secure=true`, `http-only=true` `same-stie=none`으로 설정하여 문제를 해결합니다.
- 클라이언트에서 로그인 후에 쿠키를 확인하면 정상적으로 `SameSite=None`으로 정상적으로 설정되었습니다.
![](../../images/Pasted%20image%2020260306124004.png)

