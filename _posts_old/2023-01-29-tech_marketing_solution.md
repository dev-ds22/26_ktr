---
layout: single
title: "[tech] 마케팅관련 솔루션"
excerpt: "Amplitude, Braze 데이터마케팅 솔루션, GA 버전비교"

categories:
  - tech
tags:
  - [amplitude, braze, ga, ga360]

toc: false
toc_sticky: true

date: 2023-01-29
last_modified_at: 2023-01-29
---

# Braze (CRM마켓팅 솔루션, MMA - 마켓팅 자동화 솔루션)
- 마케팅 자동화 & 개인화 솔루션. 
- 인앱(In-App), 인브라우저(In-Browser) 메시지, 푸시 메시지, 이메일, SMS, 카카오 친구톡, 알림톡 등의 멀티 채널 플랫폼에 개인화된 메시지를 보낼 수 있는 서비스 제공.
- 유저 개개인의 라이프사이클을 파악, 그 단계에 맞는 메시지 전송으로 고객 재방문과 재구매 촉진. 
- 각각의 고객에 대한 개인화된 메시지로 사람과 사람 사이의 대화처럼 느껴질 수 있다 흥보.
- 실시간 사용자 데이터를 기반으로 정교하게 타겟팅함으로써 무분별한 광고성 메시지와 차별화 
- Liquid라는 마크업 언어로 고객의 이름을 사용해서 메시지를 전송가능.
- 캠페인의 기획, 테스트, 실행까지 브레이즈의 모든 기능은 개발자의 도움 없이 마케터 혼자 대시보드에서 가능. 
- A/B 테스트 기능을 제공하여 최적화된 푸시 메시지 설정 가능.
- 고객이 앱을 삭제하였더라도 이메일이나 문자를 보내 고객들을 복귀시키게 할 수도 있음.

---

# Amplitude (이벤트 기반 사용자 행동분석 솔루션)
- 제품 분석을 위한 목적으로 만들어진 Solution으로 사용자의 행동을 Event로 정의하여 이벤트 흐름(Event stream)을 측정.

> 제품 분석은 사용자가 제품 또는 서비스에 참여하는 방식을 분석하는 프로세스. 이를 통해 제품 담당자는 사용자 참여 및 행동 데이터를 추적, 시각화 및 분석.  
> 고객을 비즈니스의 핵심에 두어 고객의 행동데이터를 분석하고, 전환 기회를 식별하고, 높은 고객 평생 가치를 가져오는 서비스경험을 만드는 프레임워크.  

- 예를 들어 우리 서비스가 웹 / 앱을 모두 운영하고 있을 때 유저들이 행동 중 하나는 웹에서 서비스의 특징을 이해하고 구매활동을 앱에서 진행가능.
- 1회의 구매 경험을 웹에서 진행한 뒤에 앱으로 넘어와서 충성고객으로 전환되는 흐름을 가져갈 수 있는데. 이러한 유저의 흐름을 우리는 아래와 같이 도식화 가능. 

![User journey와 Product Anlytics](./../../images/tech/amplitude_01.png)

- 유저의 흐름 중 각 Touch point마다의 특징을 분석하고 다음 여정으로 이어지게 하는 요소 확인가능. 
  - Sign up Touch point : 회원가입을 하게 만드는 특징을 판단
  - App install Touch point : User journey에서 wep to app을 만드는 요소 이해
  - App push Touch point : Push에 대한 반응을 통한 retention 관리, power user가 되는 aha moment 확인

### Amplitude 와 GA 비교 
- 페이지 뷰 기반의 구글 vs 이벤트 기반의 앰플리튜드
- 구글 애널리틱스는 익히 알고 있는 것처럼 페이지뷰 기반의 로그 분석 툴. 
  - 이벤트를 설정할 수 있지만 페이지뷰와 이벤트는 따로 분리된 보고서에서 분석, 두 데이터를 함께 분석할 수 없는 문제점. 
  - 전체 페이지를 렌더링하지 않고 화면을 바꾸는 네이티브 앱은 물론, SPA(싱글 페이지 어플리케이션)로 구성된 모던 웹을 트래킹하는데 단점으로 작용.

- 참고. 1

> 페이뷰가 제대로 동작을 안하고 있었는데, 이는 홈페이지가 SPA 으로 개발되었기 때문에 발생하는 문제로 확인 되었다. Google Analytics 홈페이지에서도 SPA에서 발생할 수 있는 문제에 대해 솔루션을 공지하고 있다.   
> 즉, SPA 의 경우 전통적인 웹 페이지 로드 방식과는 다르다는 것이다. 기존의 웹에서 URL 자체가 페이지의 리소스 위치를 가리키고 있었던 반면에, SPA에서는 단일 페이지 안에서 URL이 변경될 때마다, 동적으로  페이지를 생성하는 방식이다. Google Analytics 트래킹 방식이 URL에 기반하다보니 SPA에서는 제대로 동작을 하지 않았던 상황이었다.  
> **GA Developer 사이트에 제안한urlChangeTracker 플러그인을 사용하여 이를 해결**  

- 참고. 2

> 구글 애널리틱스는 페이지 뷰와 이벤트가 아예 다른 보고서에서 다뤄진다. 이로 인해, 구글에서는 페이지 뷰 기반으로 트래킹하던 구글 애널리틱스 모바일 SDK 서비스를 종료하였으며, 이를 대신하여 이벤트 기반의 파이어베이스를 연동하여 앱 분석을 하게 만들었다. 뿐만 아니라, 구글 애널리틱스 앱+웹 속성을 새로 추가하여 웹과 앱을 넘나드는 유저의 데이터를 추적할 수 있게 하였다.  
> 그러나 애초에 이벤트 기반으로 설계된 앰플리튜드에 비해 구글 애널리틱스는 한참 뒤떨어지고 있다. 당장, 위에서 얘기한 앱+웹 속성은 작년에 출시되어 아직 한참 베타 테스트 중인 서비스이며, 웹 - 구글애널리틱스, 앱 - 파이어베이스 모두 앰플리튜드에 비해 부족한 기능이 발목을 잡는다.   

- 이벤트 기반으로 설계된 앰플리튜드의 경우, 페이지 뷰는 페이지 뷰 이벤트로, 다른 행동들은 또 다른 행동 이벤트로 모두 트래킹할 수 있기 때문에, 훨씬 가변적으로 데이터를 수집할 수 있으며 이 모든 데이터를 앰플리튜드의 모든 차트에서 활용할 수 있다는 장점을 가지고 있다. 
- 구글 애널리틱스에 비해 데이터를 폭 넓게 사용할 수 있는 기반을 가지고 있다고 봐도 무방.

- 참고 사이트1 : https://blog.wiselycompany.com/wisely-amplitude
- 참고 사이트2 : https://dm-note.tistory.com/entry/google-analytics-vs-amplitude
- 참고 사이트3 : https://brunch.co.kr/@doit-dev/9

---

# Groobee (AI 개인화 상품추천 솔루션)
- 상품 기반 AI 알고리즘, 방문자 이력 기반 AI 알고리즘, 통계형 알고리즘 등 총 22가지 추천 알고리즘.
- 연관 상품, 보완 상품, 협업 필터링, 사용자 취향 기반, 구매 패턴 기반 등 구매 상황이나 페이지 특성에 맞춰 전략적 상품 추천. 
- 최적의 추천 알고리즘 자동 적용

```javascript
  // groobeeUtil
  getGroobeePrefRecommend (area, gender, age) {
    const campaignKeys = {
      local: '',
      dev: '',
      stg: '',
      prd: '',
      defalut: ''
    }
    try {
      console.log('getGroobeePrefRecommend', campaignKeys[campaignKeys] ?? campaignKeys.defalut, area, gender, age)
      return groobee.getGroobeeRecommend(campaignKeys[campaignKeys] ?? campaignKeys.defalut, { area, gender, age })
    } catch (e) {
      console.log(e)
      return null
    }
  },

  // 추천목록 받기
  getGroobeeRecommend (algorithmCd, campaignKey, goodsArray) {
    console.log('getGroobeeRecommend', algorithmCd, campaignKey, goodsArray)
    this.getGroobeeRecommendList(algorithmCd, campaignKey, goodsArray)
  },

  // 필터된 항목 중 클릭한 아이템
  setGroobeeRecommendClick (algorithmCd, campaignKey, carCd) {
    console.log('setGroobeeRecommendClick', algorithmCd, campaignKey, carCd)
    const groobeeObj = {
      algorithmCd,
      campaignKey,
      campaignTypeCd: 'RE',
      goods: [{ goodsCd: carCd }]
    }
    groobee.send('CL', groobeeObj)
  }
```

```javascript
  import { groobeeUtil } from '~/mixin/groobeeUtil'
```

---

# GA vs GA360 vs GA4 비교.

### GA(Google Analytics, 구글애널리틱스 스텐다드)

- 구글애널리틱스 무료버전으로 보통 WEB에 많이 사용.
- APP은 파이어베이스(Firebase)를 통해 데이터를 수집.

### GA360(Google Analytics360, 구글애널리틱스 프리미엄)

- 구글애널리틱스 유료버전, 무료버전과 가장 큰 차이점은 '데이터 소유권'.
  - 무료버전 : 데이터 소유권이 '구글', GA360 : 데이터 소유권이 '나'. 
- 데이터 샘플링이 없고, BigQuery를 통해 Raw Data 이용가능.
- 연간 약 1.5억의 사용료.
- APP은 파이어베이스(Firebase)를 통해 데이터 수집.

### GA4(Google Analytics v4, 구글애널리틱스4)
- 2019년 신설, WEB과 APP을 심리스(구글피셜)하게 보기위한 GA.
- GA360처럼 유료버전을 쓰지 않아도 BigQuery로 데이터를 보내주기때문에, RawData를 쿼리비용만 내고 사용가능.
- GAUI에 비해 GA4 UI에서는 많을것을 보여주지 않고, 데이터 분석을 위해 제대로 사용하기 위해선, BugQuery에 익숙해야 함.

  
---
  
# Firebase
- iOS 및 Android 모바일 개발자를 위한 Google 플랫폼
- FCM(Firebase Cloud Messaging)을 통해 PUSH 서비스 가능
  - Firebase를 이용한 PUSH 서비스를 구현할 때에는 Firebase 계정 및 Key 필요.
- GA360, GA4는 APP 데이터의 Firebase 연동을 통해 Analytics 제공
  - GA 도입시 APP 데이터의 연동을 위해서는 Firebase 계정 필요할 듯

- CRM솔루션인 BRAZE를 이용하면 이메일, 푸시, 인앱 메시지 서비스가 가능한듯 합니다.
- 참고 : http://minhyun0821.com/braze_message/



<details>
  <summary>Exp.</summary>  
  <pre>

  </pre>
</details>