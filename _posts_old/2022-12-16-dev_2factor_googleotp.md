---
layout: single
title: "[Google OTP] Spring Boot Google OTP 2단계 보안인증 "
excerpt: "Google OTP 2단계 Authenticator 개념과 간단한 코드"

categories:
  - tech
tags:
  - [tech, google otp]

toc: false
toc_sticky: true

date: 2022-12-16
last_modified_at: 2022-12-16
---
# Spring Boot Google OTP 2단계 보안인증 (Authenticator) 개념과 간단한 코드

## Spring Boot 2단계 보안인증 Google OTP, Authenticator

- 2FA (Two-factory authentication) 2단계 보안인증 :
  - 아이디, 비밀번호 로그인 다음의 2번째 인증단계
  - 이메일, 문자 메시지 또는 Google Authenticator 앱으로 전송된 6자리 코드를 입력하도록 요청
  - 발급된 코드는 30초 또는 60초 후에 만료

## Dependency

- totp, commons-codec, zxing 의존성 추가

- implementation group: 'de.taimos', name: 'totp', version: '1.0'
  - https://mvnrepository.com/artifact/de.taimos/totp

- implementation group: 'commons-codec', name: 'commons-codec', version: '1.15'
  - https://mvnrepository.com/artifact/commons-codec/commons-codec

- implementation group: 'com.google.zxing', name: 'javase', version: '3.4.1'
  - https://mvnrepository.com/artifact/com.google.zxing/javase


```gradle
  plugins {
    id 'org.springframework.boot' version '2.4.3'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
    id 'war'
  }

  group = 'com.example.opt'
  version = '0.0.1-SNAPSHOT'
  sourceCompatibility = '1.8'

  repositories {
    mavenCentral()
  }

  dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
      // 추가 
    implementation 'de.taimos:totp:1.0'
      // 추가 
    implementation 'commons-codec:commons-codec:1.10'
      // 추가 
    implementation 'com.google.zxing:javase:3.2.1'

      providedRuntime 'org.springframework.boot:spring-boot-starter-tomcat'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
  }

  test {
    useJUnitPlatform()
  }
```
 
### TOTP (Time based One-time Password)

- 시간 동기화 방식, OTP를 생성하기 위해 사용하는 입력 값으로 시간을 사용하는 방식
- HMAC 기반 일회용 암호 알고리즘(HOTP)의 확장으로 현재 시간의 고유성을 사용하여 일회용암호 생성.

>   
> Authenticator 앱을 예로 설명하면 Authenticator 앱과 서버는 같은 알고리즘을  
> 바탕으로 하기 때문에 직접적인 인증을 위한 통신이 필요하지 않습니다.  
> 작동 원리는 처음에 서버 쪽에서 해당 알고리즘으로 Key(또는 바코드 주소)를  
> 생성해주면 클라이언트는 그것을 Authenticator 앱에 입력해줍니다.  
> 그러면 앱에서는 그 Key(바코드 내부에 Key정보)를 가지고 30초마다 계속해서  
> 새로운 일회성 비밀번호를 생성합니다. 클라이언트는 앱에서 생성되는 일회용  
> 비밀번호를 서버에 입력하고, 서버에서는 그 비밀번호를 알고리즘으로 확인하는 방법.
>   
> 시간 값을 사용하기 때문에 임의의 입력값이 필요하지 않다는 점에서 사용하기  
> 간편하고, 클라이언트가 서버와 통신해야 하는 횟수가 비교적 적습니다.  
> 하지만 클라이언트와 서버의 시간 동기화가 정확하지 않으면 인증에 실패하게  
> 된다는 단점이 있으며, 이를 보완하기 위해 1~2분 정도의 OTP 생성 간격을 둠.  
>  

### commons-codec
- 입력을 16진수 및 base32로 변환하며, 최초 1회 실행되는 개인키 발급에 사용됩니다.

### zxing
- QR 코드 생성을 위한 라이브러리입니다.

### TOTPTokenGenerator

- 개인 키(비밀 키) 및 QR바코드 생성을 위한 클래스.
- Google OTP에는 base32 문자열로 인코딩 된 20바이트의 secretKey가 필요.
- generateSecretKey() 메소드는 32자의 문자열(비밀키)을 반환하며, 이 비밀키는 해당하는 회원의 google authenticator 앱을 2단계 인증에 사용될 것이기 때문에 따로 저장.

```java
  import java.io.UnsupportedEncodingException;
  import java.net.URLEncoder;
  import java.security.SecureRandom;
  import org.apache.commons.codec.binary.Base32;

  public class TOTPTokenGenerator {

    private TOTPTokenGenerator() {
    }

    private static String GOOGLE_URL = "https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=";

    // 최초 개인 Security Key 생성
    public static String generateSecretKey() {
      SecureRandom random = new SecureRandom();
      byte[] bytes = new byte[20];
      random.nextBytes(bytes);
      Base32 base32 = new Base32();
      return base32.encodeToString(bytes);
    }

    // 개인키, 계정명(시스템 사용자 ID), 발급자를 받아서 구글OTP 인증용 링크를 생성
    public static String getGoogleAuthenticatorBarcode(String secretKey, String account, String issuer) {
      try {
        return GOOGLE_URL + "otpauth://totp/"
            + URLEncoder.encode(issuer + ":" + account, "UTF-8").replace("+", "%20")
            + "?secret=" + URLEncoder.encode(secretKey, "UTF-8").replace("+", "%20")
            + "&issuer=" + URLEncoder.encode(issuer, "UTF-8").replace("+", "%20");
      } catch (UnsupportedEncodingException e) {
        throw new IllegalStateException(e);
      }
    }
  }
```

### TOTPTokenValidation

- 클라이언트로부터 입력되는 코드의 유효성을 검사하는 클래스.
- secretKey는 각각의 회원의 Authenticator 앱에서 code를 생성하기 위한 값으로 실제 로직에서는 로그인한 회원의 secretKey 값을 DB에서 가지고 와서 code 비교가능.

```java
  import org.apache.commons.codec.binary.Base32;
  import org.apache.commons.codec.binary.Hex;

  public class TOTPTokenValidation {

    private static String secretKey = "로그인한 회원에게 생성된 개인키";

    public static boolean validate(String inputCode) {
      String code = getTOTPCode();
      return code.equals(inputCode);
    }

    // OTP 검증 요청 때마다 개인키로 OTP 생성
    public static String getTOTPCode() {
      Base32 base32 = new Base32();
      // 실제로는 로그인한 회원에게 생성된 개인키가 필요합니다.
      byte[] bytes = base32.decode(TOTPTokenValidation.secretKey);
      String hexKey = Hex.encodeHexString(bytes);
      return TOTP.getOTP(hexKey);
    }
  }
```

### TOTP
- final이 붙어 더이상 확장불가
- 자신이 정의한 클래스가 잘못된 방식으로 상속, 확장되는 것을 막고자 final 키워드 사용.

```java
  import java.lang.reflect.UndeclaredThrowableException;
  import java.math.BigInteger;
  import java.security.GeneralSecurityException;
  import javax.crypto.Mac;
  import javax.crypto.spec.SecretKeySpec;

  public final class TOTP {

    private TOTP() {
      // private utility class constructor
    }

    public static String getOTP(String key) {
      return TOTP.getOTP(TOTP.getStep(), key);
    }

    private static long getStep() {
      // 30 seconds StepSize (ID TOTP)
      return System.currentTimeMillis() / 30000;
    }

    private static String getOTP(final long step, final String key) {
      String steps = Long.toHexString(step).toUpperCase();
      while (steps.length() < 16) {
        steps = "0" + steps;
      }

      // Get the HEX in a Byte[]
      final byte[] msg = TOTP.hexStr2Bytes(steps);
      final byte[] k = TOTP.hexStr2Bytes(key);

      final byte[] hash = TOTP.hmac_sha1(k, msg);

      // put selected bytes into result int
      final int offset = hash[hash.length - 1] & 0xf;
      final int binary = ((hash[offset] & 0x7f) << 24) | ((hash[offset + 1] & 0xff) << 16) | ((hash[offset + 2] & 0xff) << 8) | (hash[offset + 3] & 0xff);
      final int otp = binary % 1000000;

      String result = Integer.toString(otp);
      while (result.length() < 6) {
        result = "0" + result;
      }
      return result;
    }

    private static byte[] hexStr2Bytes(final String hex) {
      // Adding one byte to get the right conversion
      // values starting with "0" can be converted
      final byte[] bArray = new BigInteger("10" + hex, 16).toByteArray();
      final byte[] ret = new byte[bArray.length - 1];

      // Copy all the REAL bytes, not the "first"
      System.arraycopy(bArray, 1, ret, 0, ret.length);
      return ret;
    }

    private static byte[] hmac_sha1(final byte[] keyBytes, final byte[] text) {
      try {
        final Mac hmac = Mac.getInstance("HmacSHA1");
        final SecretKeySpec macKey = new SecretKeySpec(keyBytes, "RAW");
        hmac.init(macKey);
        return hmac.doFinal(text);
      } catch (final GeneralSecurityException gse) {
        throw new UndeclaredThrowableException(gse);
      }
    }
  }
```

### AuthenticatorApplication

- 바코드 url이 아닌 실제 바코드 이미지와 secretKey 값을 클라이언트에게 표시 
- 클라이언트는 바코드를 찍어서 등록하거나 key를 직접 등록하는 방법 사용가능.

```java
import com.example.authenticator.api.TOTPTokenGenerator;
import com.example.authenticator.api.TOTPTokenValidation;
import java.util.Scanner;

public class AuthenticatorApplication {

  public static void main(String[] args) {
    generateSecurityKey();
    validAuthenticatorCode();
  }

  private static void validAuthenticatorCode() {
    Scanner scanner = new Scanner(System.in);
    String code = scanner.nextLine();
    if (TOTPTokenValidation.validate(code)) {
      System.out.println("Logged in successfully");
    } else {
      System.out.println("Invalid 2FA Code");
    }
  }

  private static void generateSecurityKey() {
    // secretKey 생성
    String secretKey = TOTPTokenGenerator.generateSecretKey();
    System.out.println(secretKey);
    String account = "otptest@google.com";
    String issuer = "otpTest";
    // secretKey + account + issuer => QR 바코드 생성
    String barcodeUrl = TOTPTokenGenerator.getGoogleAuthenticatorBarcode(secretKey, account, issuer);
    System.out.println(barcodeUrl);
  }
}
```

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>