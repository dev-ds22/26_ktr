---
layout: single
title: "Apple Login  단계 별 추적"

categories:
  - tech
tags:
  - tech

toc: false
toc_sticky: true

date: 2026-05-13
last_modified_at: 2026-05-13 17:35:07
---

## 예제
```java
URL requestUrl = new URL("https://appleid.apple.com/auth/token");
conn = (HttpURLConnection) requestUrl.openConnection();
conn.setRequestMethod("POST");
conn.setRequestProperty("Content-Type", "application/x-www-form-urlencoded")
conn.setDoOutput(true);
// 3. 바디 전송
try (OutputStream os = conn.getOutputStream()) {
  os.write(postDataBytes);
  os.flush();
}
// 4. 응답 코드 확인
int statusCode = conn.getResponseCode();
if (statusCode != 200) {
  return json.toString();
}
// 5. 응답 읽기
reader = new BufferedReader(new InputStreamReader(conn.getInputStream(), StandardCharsets.UTF_8));
String line = reader.readLine();
while (line != null) {
  result.append(line);
  line=reader.readLine()
}
// 6. JSON 파싱
json = new JSONObject(result.toString());
// 7. id_token 파싱 (3.10 기준)
if (json.has("id_token")) {
  String idToken = json.getString("id_token");
  SignedJWT signedJWT = SignedJWT.parse(idToken);
  // getJWTClaimsSet() 없음 → payload에서 직접 claims 추출
  JSONObject claimsJson = new JSONObject(signedJWT.getPayload().toJSONObject());
  // 원래 토큰 응답 + 파싱 결과 합치기
  json.put("claims", claimsJson);
}
return json.toString();
```

정확도: 92%

# Apple `/auth/token` 응답 기준 Java 리턴 예제

## 1. 현재 Apple Token API 응답 구조

Apple Sign in with Apple의 token endpoint는 아래 URL입니다.

```text
https://appleid.apple.com/auth/token
```

Apple 공식 문서 기준으로 authorization code 검증 또는 refresh token 검증에 사용하며, 응답에는 일반적으로 `access_token`, `token_type`, `expires_in`, `refresh_token`, `id_token`이 포함될 수 있습니다. `refresh_token`은 최초 authorization code 교환 시 내려오며, 이후 항상 내려온다고 보면 안 됩니다. ([Apple Developer](https://developer.apple.com/documentation/signinwithapplerestapi/generate-and-validate-tokens?utm_source=chatgpt.com "Token validation | Apple Developer Documentation"))

## 2. 정상 응답 예시

```json
{
  "access_token": "ACCESS_TOKEN_VALUE",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "REFRESH_TOKEN_VALUE",
  "id_token": "JWT_ID_TOKEN_VALUE"
}
```

## 3. `id_token` Claims 예시

`id_token`은 JWT 형식입니다. 파싱하면 보통 아래와 같은 claim을 확인합니다.

```json
{
  "iss": "https://appleid.apple.com",
  "aud": "com.example.service",
  "exp": 1710000000,
  "iat": 1709996400,
  "sub": "000123.abcdef...",
  "email": "user@example.com",
  "email_verified": "true",
  "is_private_email": "false",
  "auth_time": 1709996400,
  "nonce_supported": true
}
```

주의할 점은 `email`, `email_verified`, `is_private_email`은 상황에 따라 없을 수 있습니다. Apple 로그인에서 사용자 식별의 기본 키는 `email`보다 `sub`를 기준으로 잡는 것이 안전합니다.

## 4. 권장 리턴 JSON 구조

기존 코드처럼 Apple 원 응답에 `claims`를 추가하는 방식은 실무적으로 괜찮습니다.

```json
{
  "success": true,
  "statusCode": 200,
  "access_token": "ACCESS_TOKEN_VALUE",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "REFRESH_TOKEN_VALUE",
  "id_token": "JWT_ID_TOKEN_VALUE",
  "claims": {
    "iss": "https://appleid.apple.com",
    "aud": "com.example.service",
    "sub": "000123.abcdef...",
    "email": "user@example.com",
    "email_verified": "true",
    "is_private_email": "false",
    "exp": 1710000000,
    "iat": 1709996400
  }
}
```

## 5. 수정된 Java 예제

아래 예제는 질문 코드 흐름을 유지하면서, 현재 Apple token API 응답에 맞게 리턴값을 구성한 예시입니다.

```java
import com.nimbusds.jwt.SignedJWT;
import org.json.JSONObject;

import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.nio.charset.StandardCharsets;

public String requestAppleToken(byte[] postDataBytes) {
    HttpURLConnection conn = null;

    JSONObject responseJson = new JSONObject();

    try {
        URL requestUrl = new URL("https://appleid.apple.com/auth/token");

        conn = (HttpURLConnection) requestUrl.openConnection();
        conn.setRequestMethod("POST");
        conn.setRequestProperty("Content-Type", "application/x-www-form-urlencoded");
        conn.setRequestProperty("Accept", "application/json");
        conn.setConnectTimeout(5000);
        conn.setReadTimeout(10000);
        conn.setDoOutput(true);

        try (OutputStream os = conn.getOutputStream()) {
            os.write(postDataBytes);
            os.flush();
        }

        int statusCode = conn.getResponseCode();
        responseJson.put("statusCode", statusCode);

        InputStream responseStream = statusCode >= 200 && statusCode < 300
                ? conn.getInputStream()
                : conn.getErrorStream();

        StringBuilder result = new StringBuilder();

        if (responseStream != null) {
            try (BufferedReader reader = new BufferedReader(
                    new InputStreamReader(responseStream, StandardCharsets.UTF_8))) {

                String line;
                while ((line = reader.readLine()) != null) {
                    result.append(line);
                }
            }
        }

        if (result.length() == 0) {
            responseJson.put("success", false);
            responseJson.put("error", "EMPTY_RESPONSE");
            return responseJson.toString();
        }

        JSONObject appleJson = new JSONObject(result.toString());

        if (statusCode != 200) {
            appleJson.put("success", false);
            appleJson.put("statusCode", statusCode);
            return appleJson.toString();
        }

        appleJson.put("success", true);
        appleJson.put("statusCode", statusCode);

        if (appleJson.has("id_token") && !appleJson.isNull("id_token")) {
            String idToken = appleJson.getString("id_token");

            SignedJWT signedJWT = SignedJWT.parse(idToken);

            JSONObject claimsJson = new JSONObject(signedJWT.getJWTClaimsSet().toJSONObject());

            appleJson.put("claims", claimsJson);

            appleJson.put("appleUserId", claimsJson.optString("sub", null));
            appleJson.put("email", claimsJson.optString("email", null));
            appleJson.put("emailVerified", claimsJson.opt("email_verified"));
            appleJson.put("privateEmail", claimsJson.opt("is_private_email"));
        }

        return appleJson.toString();

    } catch (Exception e) {
        responseJson.put("success", false);
        responseJson.put("error", e.getClass().getSimpleName());
        responseJson.put("message", e.getMessage());
        return responseJson.toString();

    } finally {
        if (conn != null) {
            conn.disconnect();
        }
    }
}
```

## 6. `getJWTClaimsSet()`이 없는 경우

질문 코드에 적힌 것처럼 사용 중인 Nimbus JOSE JWT 버전에서 `getJWTClaimsSet()` 사용이 어렵다면 아래 방식으로 처리할 수 있습니다.

```java
SignedJWT signedJWT = SignedJWT.parse(idToken);

JSONObject claimsJson = new JSONObject(
        signedJWT.getPayload().toJSONObject()
);
```

따라서 해당 부분만 아래처럼 바꾸면 됩니다.

```java
if (appleJson.has("id_token") && !appleJson.isNull("id_token")) {
    String idToken = appleJson.getString("id_token");

    SignedJWT signedJWT = SignedJWT.parse(idToken);

    JSONObject claimsJson = new JSONObject(
            signedJWT.getPayload().toJSONObject()
    );

    appleJson.put("claims", claimsJson);

    appleJson.put("appleUserId", claimsJson.optString("sub", null));
    appleJson.put("email", claimsJson.optString("email", null));
    appleJson.put("emailVerified", claimsJson.opt("email_verified"));
    appleJson.put("privateEmail", claimsJson.opt("is_private_email"));
}
```

## 7. Apple Token 요청 Body 예시

`postDataBytes`는 보통 아래 값을 `application/x-www-form-urlencoded`로 만든 결과입니다.

```text
client_id=CLIENT_ID
client_secret=CLIENT_SECRET_JWT
code=AUTHORIZATION_CODE
grant_type=authorization_code
redirect_uri=REDIRECT_URI
```

Java 구성 예시는 아래와 같습니다.

```java
String postData =
        "client_id=" + URLEncoder.encode(clientId, StandardCharsets.UTF_8.name()) +
        "&client_secret=" + URLEncoder.encode(clientSecret, StandardCharsets.UTF_8.name()) +
        "&code=" + URLEncoder.encode(code, StandardCharsets.UTF_8.name()) +
        "&grant_type=authorization_code" +
        "&redirect_uri=" + URLEncoder.encode(redirectUri, StandardCharsets.UTF_8.name());

byte[] postDataBytes = postData.getBytes(StandardCharsets.UTF_8);
```

`refresh_token`으로 갱신할 때는 body가 달라집니다.

```text
client_id=CLIENT_ID
client_secret=CLIENT_SECRET_JWT
grant_type=refresh_token
refresh_token=REFRESH_TOKEN
```

## 8. 에러 응답 예시

Apple token API 실패 시에는 일반적으로 JSON 에러가 내려옵니다.

```json
{
  "error": "invalid_grant"
}
```

위 예제 코드에서는 실패 시 아래처럼 리턴됩니다.

```json
{
  "error": "invalid_grant",
  "success": false,
  "statusCode": 400
}
```

## 9. 실무상 반드시 추가 검증해야 하는 항목

현재 코드는 `id_token`을 **파싱만** 합니다. 운영 로그인 검증으로는 부족합니다.

|검증|설명|
|---|---|
|JWT 서명 검증|Apple 공개키로 `id_token` 서명 검증 필요|
|`iss`|`https://appleid.apple.com`인지 확인|
|`aud`|내 `client_id`와 일치하는지 확인|
|`exp`|만료되지 않았는지 확인|
|`sub`|사용자 고유 식별자로 저장|
|`nonce`|로그인 요청 시 nonce를 썼다면 일치 여부 확인|

Apple 로그인에서는 `/auth/token` 응답을 받은 뒤 identity token을 검증하고 authorization code를 검증하는 흐름이 요구됩니다. Apple Developer Forum의 공식 답변에서도 신규 계정 생성 시 identity token과 authorization code를 서버로 안전하게 전송하고, `/auth/token` endpoint로 authorization code를 검증한 뒤 token response를 안전하게 저장하라고 안내합니다. ([Apple Developer](https://developer.apple.com/forums/tags/sign-in-with-apple?utm_source=chatgpt.com "Sign in with Apple | Apple Developer Forums"))

## 10. 최종 리턴 예제

성공 시:

```json
{
  "success": true,
  "statusCode": 200,
  "access_token": "ACCESS_TOKEN_VALUE",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "REFRESH_TOKEN_VALUE",
  "id_token": "JWT_ID_TOKEN_VALUE",
  "claims": {
    "iss": "https://appleid.apple.com",
    "aud": "com.example.service",
    "sub": "000123.abcdef...",
    "email": "user@example.com",
    "email_verified": "true",
    "is_private_email": "false",
    "exp": 1710000000,
    "iat": 1709996400
  },
  "appleUserId": "000123.abcdef...",
  "email": "user@example.com",
  "emailVerified": "true",
  "privateEmail": "false"
}
```

실패 시:

```json
{
  "success": false,
  "statusCode": 400,
  "error": "invalid_grant"
}
```

## 핵심 수정 포인트

```text
1. statusCode != 200일 때도 errorStream을 읽어 Apple 에러 JSON을 반환한다.
2. 정상 응답에는 success, statusCode를 추가한다.
3. id_token이 있으면 JWT payload claims를 추가한다.
4. sub를 appleUserId로 별도 추출한다.
5. email은 없을 수 있으므로 optString 또는 opt 사용한다.
6. 운영 검증에서는 JWT 파싱만 하지 말고 Apple 공개키 기반 서명 검증까지 해야 한다.
```



# `JSONObject["id_token"] not found` 원인과 대책

## 1. 결론

아래 오류는 **`apiResponse` JSON 안에 `id_token` 필드가 없는데 `getString("id_token")`을 호출했기 때문에 발생**한 오류입니다.

```text
org.json.JSONException: JSONObject["id_token"] not found.
```

즉, 문제 지점은 아래 코드입니다.

```java
JSONObject tokenResponse = new JSONObject(apiResponse);
String idToken = tokenResponse.getString("id_token");
```

`org.json.JSONObject#getString()`은 해당 key가 없으면 `JSONException`을 발생시킵니다. 따라서 Apple 응답에서 `id_token`이 없을 수 있는 경우를 반드시 방어해야 합니다.

---

## 2. 가장 가능성 높은 원인

|구분|원인|설명|
|---|---|---|
|1|Apple token API 실패 응답|`{"error":"invalid_grant"}` 같은 응답에는 `id_token`이 없음|
|2|HTTP 200이 아닌 응답을 그대로 파싱|실패 응답인데도 정상 응답처럼 `id_token`을 꺼냄|
|3|`grant_type=refresh_token` 요청|refresh token 갱신 응답에는 `id_token`이 항상 있다고 보면 안 됨|
|4|`authorization_code` 오류|code 만료, 재사용, client_id 불일치 등|
|5|`client_secret` 오류|Team ID, Key ID, Service ID/Bundle ID, 서명 알고리즘 오류|
|6|`redirect_uri` 불일치|최초 authorize 요청의 redirect_uri와 token 요청의 redirect_uri 불일치|
|7|응답 로깅 미흡|실제 Apple 응답이 무엇인지 확인하지 않고 `id_token`만 꺼냄|

Apple의 Sign in with Apple token endpoint는 `authorization_code` 또는 `refresh_token`을 검증하는 endpoint이며, 정상적인 authorization code 교환 응답에는 token response가 내려오지만, 실패 시에는 `error` 형태의 응답이 내려옵니다. Apple은 `invalid_grant`가 authorization grant 또는 refresh token 문제에서 발생할 수 있다고 설명합니다. ([Apple Developer](https://developer.apple.com/documentation/technotes/tn3107-resolving-sign-in-with-apple-response-errors?utm_source=chatgpt.com "TN3107: Resolving Sign in with Apple response errors"))

---

## 3. 우선 실제 `apiResponse`를 먼저 확인해야 함

현재 오류 로그만 보면 `id_token`이 없다는 것만 알 수 있고, **왜 없는지는 `apiResponse` 원문을 봐야 합니다.**

반드시 아래 로그를 임시로 추가하십시오.

```java
log.info("[Apple Token Response] {}", apiResponse);
```

다만 운영에서는 `access_token`, `refresh_token`, `id_token` 전체를 장기간 로그로 남기면 안 됩니다. 장애 확인 후 제거하거나 마스킹해야 합니다.

권장 임시 로그:

```java
JSONObject tokenResponse = new JSONObject(apiResponse);

log.info("[Apple Token Response keys] {}", tokenResponse.keySet());
log.info("[Apple Token Response error] {}", tokenResponse.optString("error", ""));
log.info("[Apple Token Response error_description] {}", tokenResponse.optString("error_description", ""));
```

---

## 4. 즉시 수정해야 할 코드

### 4.1 문제 코드

```java
JSONObject tokenResponse = new JSONObject(apiResponse);
String idToken = tokenResponse.getString("id_token");
```

이 코드는 `id_token`이 없으면 바로 예외가 발생합니다.

### 4.2 방어 코드

```java
JSONObject tokenResponse = new JSONObject(apiResponse);

if (tokenResponse.has("error")) {
    String error = tokenResponse.optString("error");
    String errorDescription = tokenResponse.optString("error_description");

    throw new MmBizException(
        "Apple token API failed. error=" + error + ", description=" + errorDescription
    );
}

String idToken = tokenResponse.optString("id_token", null);

if (idToken == null || idToken.isEmpty()) {
    throw new MmBizException("Apple token response does not contain id_token. response keys=" + tokenResponse.keySet());
}
```

`getString()` 대신 `optString()` 또는 `has()` 검사를 사용해야 합니다.

---

## 5. 권장 처리 구조

Apple token API 응답은 아래 2가지로 나누어 처리해야 합니다.

### 정상 응답

```json
{
  "access_token": "ACCESS_TOKEN_VALUE",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "REFRESH_TOKEN_VALUE",
  "id_token": "JWT_ID_TOKEN_VALUE"
}
```

### 실패 응답

```json
{
  "error": "invalid_grant",
  "error_description": "The code has expired or has been revoked."
}
```

따라서 파싱 순서는 아래가 안전합니다.

```text
1. HTTP statusCode 확인
2. 응답 body 원문 읽기
3. JSON 파싱
4. error 존재 여부 확인
5. id_token 존재 여부 확인
6. id_token JWT 파싱
7. iss/aud/exp/sub 검증
```

---

## 6. 개선된 Java 예제

```java
JSONObject tokenResponse = new JSONObject(apiResponse);

// 1. Apple 에러 응답 우선 확인
if (tokenResponse.has("error")) {
    String error = tokenResponse.optString("error", "");
    String errorDescription = tokenResponse.optString("error_description", "");

    log.error("[Apple Token Error] error={}, description={}", error, errorDescription);

    throw new MmBizException("Apple token API error: " + error);
}

// 2. id_token 존재 여부 확인
String idToken = tokenResponse.optString("id_token", "");

if (idToken.isEmpty()) {
    log.error("[Apple Token Error] id_token not found. responseKeys={}", tokenResponse.keySet());

    throw new MmBizException("Apple id_token not found.");
}

// 3. id_token 파싱
SignedJWT signedJWT = SignedJWT.parse(idToken);

JSONObject claimsJson = new JSONObject(
        signedJWT.getPayload().toJSONObject()
);

// 4. 필수 claim 확인
String issuer = claimsJson.optString("iss", "");
String audience = claimsJson.optString("aud", "");
String appleUserId = claimsJson.optString("sub", "");

if (!"https://appleid.apple.com".equals(issuer)) {
    throw new MmBizException("Invalid Apple id_token issuer.");
}

if (!EXPECTED_CLIENT_ID.equals(audience)) {
    throw new MmBizException("Invalid Apple id_token audience.");
}

if (appleUserId.isEmpty()) {
    throw new MmBizException("Apple user id(sub) not found.");
}

// 5. 결과 구성
tokenResponse.put("claims", claimsJson);
tokenResponse.put("appleUserId", appleUserId);
tokenResponse.put("email", claimsJson.optString("email", ""));

return tokenResponse.toString();
```

---

## 7. HTTP 응답 처리도 같이 수정 필요

이전 코드에서 `statusCode != 200`일 때 `json.toString()`을 반환하면, 실제 Apple error body를 놓칠 수 있습니다.

### 비권장

```java
int statusCode = conn.getResponseCode();

if (statusCode != 200) {
    return json.toString();
}
```

이러면 `json`이 비어 있거나 이전 값일 수 있습니다.

### 권장

```java
int statusCode = conn.getResponseCode();

InputStream responseStream = statusCode >= 200 && statusCode < 300
        ? conn.getInputStream()
        : conn.getErrorStream();

StringBuilder result = new StringBuilder();

try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(responseStream, StandardCharsets.UTF_8))) {

    String line;
    while ((line = reader.readLine()) != null) {
        result.append(line);
    }
}

JSONObject tokenResponse = new JSONObject(result.toString());
tokenResponse.put("statusCode", statusCode);

if (statusCode != 200) {
    tokenResponse.put("success", false);
    return tokenResponse.toString();
}

tokenResponse.put("success", true);
```

이렇게 해야 Apple이 내려준 `invalid_grant`, `invalid_client` 같은 실제 원인을 확인할 수 있습니다.

---

## 8. `id_token`이 없는 대표 Apple 응답별 원인

|Apple 응답|가능 원인|조치|
|---|---|---|
|`invalid_grant`|code 만료, code 재사용, client_id 불일치, redirect_uri 불일치|authorization code 새로 발급, client_id/redirect_uri 확인|
|`invalid_client`|client_secret JWT 오류, Team ID/Key ID/Service ID 오류|client_secret 생성 로직 확인|
|`invalid_request`|필수 파라미터 누락|`client_id`, `client_secret`, `code`, `grant_type`, `redirect_uri` 확인|
|`unsupported_grant_type`|grant_type 오류|`authorization_code` 또는 `refresh_token` 확인|
|빈 응답|네트워크/SSL/프록시 문제|statusCode, errorStream, timeout 로그 확인|

Apple 개발자 포럼 사례에서도 token 교환 단계에서 `invalid_client`, `invalid_grant`가 발생할 수 있으며, client_id, client_secret, code, grant_type, redirect_uri 조합을 확인하는 것이 핵심입니다. ([Apple Developer](https://developer.apple.com/forums/tags/sign-in-with-apple-js?utm_source=chatgpt.com "Sign in with Apple JS | Apple Developer Forums"))

---

## 9. 특히 많이 발생하는 실수

### 9.1 Authorization Code 재사용

Apple authorization code는 1회성입니다.  
한 번 `/auth/token`에서 사용한 code를 다시 사용하면 `invalid_grant`가 날 수 있습니다.

```text
같은 code로 재시도 → invalid_grant → id_token 없음 → getString("id_token") 예외
```

### 9.2 `client_id` 불일치

iOS Native 로그인과 Web 로그인에서 `client_id`가 달라질 수 있습니다.

|로그인 방식|client_id|
|---|---|
|iOS Native 앱|Bundle ID|
|Web / Android / WebView|Service ID|

Apple 응답에 `client_id mismatch`가 포함되는 사례도 있으며, authorization code가 발급된 client_id와 token 요청의 client_id가 다르면 `invalid_grant`가 발생할 수 있습니다. ([Velog](https://velog.io/%40ncookie/%EC%95%A0%ED%94%8C-%EB%A1%9C%EA%B7%B8%EC%9D%B8-Invalidgrant-%EC%97%90%EB%9F%AC?utm_source=chatgpt.com "[IMAD] 애플 로그인 Invalid_grant 에러"))

### 9.3 `redirect_uri` 불일치

최초 authorize 요청에 `redirect_uri`를 넣었다면 token 요청에도 같은 값을 넣어야 합니다.

```text
authorize 요청 redirect_uri
≠
token 요청 redirect_uri
→ invalid_grant 가능
```

### 9.4 `identityToken`과 `authorizationCode` 혼동

클라이언트에서 서버로 보낼 때 `identityToken`을 `code`로 잘못 보내면 token endpoint에서 실패합니다.

|값|용도|
|---|---|
|`authorizationCode`|`/auth/token`의 `code`로 전달|
|`identityToken`|JWT 검증용|
|`user`|Apple user identifier|
|`email`|최초 동의 시에만 내려올 수 있음|

Auth0 커뮤니티 사례에서도 `userToken`을 authorization code로 잘못 사용해 `invalid_grant`가 발생한 사례가 보고되어 있습니다. ([community.auth0.com](https://community.auth0.com/t/sign-in-with-apple-fails-with-invalid-grant/52407?utm_source=chatgpt.com "Sign In With Apple Fails with \"Invalid Grant\""))

---

## 10. 현재 코드에 대한 직접 대책

현재 오류 위치:

```java
BuyerLoginController.oauth_apple(BuyerLoginController.java:370)
tokenResponse.getString("id_token")
```

이 라인은 아래처럼 바꾸는 것이 맞습니다.

```java
String idToken = tokenResponse.optString("id_token", null);

if (idToken == null || idToken.isEmpty()) {
    log.error("[Apple Login] id_token not found. tokenResponse={}", maskAppleTokenResponse(tokenResponse));
    throw new MmBizException("Apple id_token not found.");
}
```

마스킹 함수 예시:

```java
private String maskAppleTokenResponse(JSONObject json) {
    JSONObject copy = new JSONObject(json.toString());

    if (copy.has("access_token")) {
        copy.put("access_token", "***");
    }

    if (copy.has("refresh_token")) {
        copy.put("refresh_token", "***");
    }

    if (copy.has("id_token")) {
        copy.put("id_token", "***");
    }

    return copy.toString();
}
```

---

## 11. 최종 권장 코드 구조

```java
JSONObject tokenResponse = new JSONObject(apiResponse);

// 실패 응답 처리
if (tokenResponse.has("error")) {
    log.error("[Apple Login] token API failed. response={}", tokenResponse.toString());
    throw new MmBizException("Apple token API failed: " + tokenResponse.optString("error"));
}

// id_token 필수 확인
String idToken = tokenResponse.optString("id_token", "");

if (idToken.isEmpty()) {
    log.error("[Apple Login] id_token missing. responseKeys={}", tokenResponse.keySet());
    throw new MmBizException("Apple id_token missing.");
}

// JWT payload 파싱
SignedJWT signedJWT = SignedJWT.parse(idToken);
JSONObject claimsJson = new JSONObject(signedJWT.getPayload().toJSONObject());

// 필수 claim
String iss = claimsJson.optString("iss", "");
String aud = claimsJson.optString("aud", "");
String sub = claimsJson.optString("sub", "");

if (!"https://appleid.apple.com".equals(iss)) {
    throw new MmBizException("Invalid Apple issuer.");
}

if (!EXPECTED_CLIENT_ID.equals(aud)) {
    throw new MmBizException("Invalid Apple audience.");
}

if (sub.isEmpty()) {
    throw new MmBizException("Apple sub missing.");
}

tokenResponse.put("claims", claimsJson);
tokenResponse.put("appleUserId", sub);

return tokenResponse.toString();
```

---

## 12. 운영 점검 체크리스트

-  `apiResponse` 원문에 `error`가 있는지 확인
    
-  HTTP statusCode가 200인지 확인
    
-  실패 시 `conn.getErrorStream()`을 읽고 있는지 확인
    
-  `grant_type=authorization_code`인지 확인
    
-  `code`가 실제 authorization code인지 확인
    
-  같은 code를 두 번 사용하지 않는지 확인
    
-  iOS Native면 `client_id=Bundle ID`인지 확인
    
-  Web/WebView/Android면 `client_id=Service ID`인지 확인
    
-  `client_secret`의 `sub`가 `client_id`와 같은지 확인
    
-  `client_secret`의 `iss=Team ID`, `kid=Key ID`, `aud=https://appleid.apple.com`인지 확인
    
-  `redirect_uri`가 최초 authorize 요청과 token 요청에서 같은지 확인
    

---

## 핵심 요약

```text
오류 원인:
Apple 응답 JSON 안에 id_token이 없는데 getString("id_token")을 호출했기 때문.

가장 흔한 실제 원인:
Apple token API가 정상 응답이 아니라 invalid_grant, invalid_client 같은 error 응답을 반환함.

대책:
getString()을 바로 호출하지 말고, statusCode/error/id_token 존재 여부를 먼저 확인한다.
HTTP 200이 아닐 때는 getErrorStream()을 읽어 Apple의 실제 오류 JSON을 반환한다.
authorizationCode, client_id, client_secret, redirect_uri 불일치를 점검한다.
```

현재 로그만 기준으로는 **`apiResponse`가 정상 token response가 아닌데 정상 응답처럼 처리한 것**이 가장 유력합니다.


```bash
curl -v -X POST "https://appleid.apple.com/auth/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=org.buykorea.applelogin" \
  -d "client_secret=eyJraWQiOiJMR1Q2TUpSWjk0IiwiYWxnIjoiRVMyNTYifQ.eyJzdWIiOiJvcmcuYnV5a29yZWEubG9naW5hcHBsZSIsImF1ZCI6Imh0dHBzOlwvXC9hcHBsZWlkLmFwcGxlLmNvbSIsImlzcyI6IlpZWVYzMjlXMzQiLCJleHAiOjE3NzgxMjU1MDAsImlhdCI6MTc3ODEyMTkwMH0.FGVp3_513q9dlAFN2o4mINBdUvNWO_VlspFzoCcdiPYgn_Hb4PoQvxv3hYLmorprGTdjYJQGMvcxtm2anY5AjA" \
  -d "grant_type=authorization_code" \
  -d "refresh_token="
  
```

<details>
  <summary>참고</summary>  
  <pre>

  </pre>
</details>
