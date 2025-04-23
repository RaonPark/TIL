# JWT(Json Web Token)에 대해 알아보자.

## JWT의 구조
- header
  - alg: 토큰을 서명하기 위한 알고리즘이다. HMAC SHA-256등을 사용한다.
  - typ: 토큰의 타입이다. JWT인 경우에는 jwt를 사용한다.
- payload: 토큰의 바디에 해당한다. 이 때, payload는 암호화되지 않으므로 누구나 다 볼 수 있다는 것이 특징이다. 따라서 credentials와 같은 민감한 정보를 포함하지 않는 것이 좋다.
  - claim: 세 개의 클레임이 존재하며, Registered Claim, Public Claim, Private Claim이 존재하고, 실제 데이터가 들어있는 부분이라고 하면 될 것이다.
    - Registered Claim
      - iss: issuer
      - exp: expiration time
      - iat: issued at
    - Public Claim 
      - 충돌이 발생할 수 있기 때문에 Public Name: value 이런 형식으로 하던가 해야한다.
    - Private Claim
      - Register Claim이나 Public Claim에 없는 Private names: names 형태의 key-value 형태이다.

## Access Token과 Refresh Token
Access Token으로 서버와 소통이 가능하다. 그리고 Refresh Token을 사용하여 Access Token의 리프레시가 가능하다. 만료된 Access Token을 Refresh Token으로 업데이트한다.

## Jwt의 문제점과 해결법
Jwt는 탈취당할 수 있다. 다음과 같은 방식으로 해결할 수 있다.
- Jwt를 Cookie에 넣어둔다. 이는 XSS 공격을 무력화할 수 있으며, HttpOnly를 해둬서 JS로 토큰에 접근하는 것을 막을 수 있다.

