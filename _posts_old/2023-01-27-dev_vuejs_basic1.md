---
layout: single
title: "[vuejs] Vue.js 프로젝트 기본 구성"
excerpt: "Vue.js 프로젝트 기본 구성"

categories:
  - tech
tags:
  - [vuejs]

toc: false
toc_sticky: true

date: 2023-01-27
last_modified_at: 2023-01-27
---

# Vue.js 프로젝트 기본 구성

원본 : https://ospace.tistory.com/783

## 1. 초기화

- Vue.js 버전2 기준.
- vue cli 설치.

```
  npm install -g @vue/cli-init
```

- 프로젝트 이름은 foo

```
  vue create foo
```

- Vue 2 선택.

```
? Please pick a preset: (Use arrow keys)
> Default ([Vue 2] babel, eslint)
  Default (Vue 3) ([Vue 3] babel, eslint)
  Manually select features
```

- 아래 형태의 폴더 구조와 파일이 생성.

```
  foo/
  + node_modules/
  * public/
    + favicon.ico
    * index.html
  + src/
    + assets/
      + logo.png
    + components/
      + HelloWorld.vue
    + App.vue
    + main.js
```

- 실행, 테스트

```
  yarn serve
```

```
  App running at:
  - Local:   http://localhost:8080/
  - Network: http://x.x.x.x:8080/

  Note that the development build is not optimized.
  To create a production build, run yarn build.
```

- Note: 빌드할 경우 vue-cli 2.x로 생성된 프로젝트는 publicPath가 "/dist"로 인식되서 빌드된 파일과 섞이게 된다. 
- vue-cli 3.x : "/public" 폴더가 별도로 존재하며 "/dist"와 분리된 구조.

## 2. Route 구성

### 설치

- 패키지를 설치.

```
  yarn add vue-router
```

### index.js 파일 추가

- 간단한 router을 생성 
  - 프로젝트 최상위 폴더에 있는 src 폴더에 routes 폴더를 생성
  - 아래 index.js 파일을 routes 폴더에 생성.

```javascript
  import Vue from 'vue';
  import VueRouter from 'vue-router';

  Vue.use(VueRouter);
  const router = new VueRouter({
      mode: 'history',
      routes: [
          {
              path: '/',
              name: 'home',
              component: ()=> import('@/Home.vue'),
          },
          {
              path: '/bar',
              name: 'bar',
              component: ()=> import('@/components/Bar.vue'),
          },
      ],
  });
  export default router;
```

### main.js 파일에 라우터 등록

- route 폴더에 생성된 index.js을 활성화하기 위해서는 main.js 파일에 이하 내용 추가.

```javascript
  import router from './routes'

  ...
  new Vue({
      ...
      router,
  }).$mount('#app')
```

- router 이름은 변경불가.

### App.vue 파일 변경

- 기존 페이지를 홈페이지로 만들고 링크 클릭했을 때에 화면 전환.
  - 앞에 라우터 설정에서 "/" 경로에 대한 component을 Home.vue로 지정(루트 페이지로 Home.vue 로딩).
- 기존 App.vue을 Home.vue로 파일명을 변경.

```html
  <template>
      <p>
        <router-link to="/">Home</router-link> / 
        <router-link to="/bar">Go to Bar</router-link>
        <router-view></router-view>
      </p>
  </template>
```

- router-view 부분이 라우터의 component내용으로 대처.
- 최상단 App.vue가 기본으로 로딩되며 컴포넌트가 배치되고 라우터 설정에 따라 App.vue에 로딩될 컴포넌트가 지정되며 해당 컴포넌트는 router-view에 표시.
- router-link은 navigator역활로 a 태그와 동일한 기능. router-link은 현재 라우터($router)와 경로가 일치할 경우 router-link-active 클래스가 활성화.

### Bar.vue 파일 추가

- 테스트용 Bar.vue 파일을 src/components/ 폴더에 추가.

```html
  <template>
      <h1>Bar Page</h1>
  </template>
```

- 실행해보면, 화면 상단에 "Home / Go to Bar" 네비게이션 표시. 링크클릭 시 Home.vue 또는 Bar.vue 페이지가 표시. 
- 라우터 기능을 활용하여 Header, Side, Body, GNB(Global Navigation Bar), LNB(Local Navigation Bar) 등 페이지를 구성하는 레이아웃 작성 가능

### Proxy 구성

- 로컬에서 개발을 할 경우 API 서버로 호출을 위해 Proxy 사용. 
- 특정 경로에 대한 요청을 지정된 API 서버로 전달해주는 기능. 
- 프로젝트 루트에 vue.config.js파일을 생성하고 이하내용 추가.

```javascript
  module.exports = {
    devServer: {
      proxy: {
        '/api': {
          target: 'API서버 주소',
          changeOrigin: true,
          exposedHeaders: ['Content-Disposition'],
        },
      },
    },
  };
```

- "/api"가 시작하는 모든 요청은 target으로 전송. 
  - 추후 웹서버로 구성해서 페이지를 구동할 경우 Proxy 설정에 적용될 내용.

## 3. Axio 사용하여 API 호출
- axio는 vue.js에서 API 서버로 호출 때에 사용하는 라이브러리.
- src 폴더에 api폴더를 생성하고 그 안에 index.js 파일을 생성.

```javascript
  import axios from 'axios';

  const instance = axios.create();
  instance.defaults.timeout = 2500; // 기본 타임아웃
  instance.get('/foo', { timeout: 5000 }); // 특정 API에 대한 타임아웃
  instance.defaults.headers.common['Authorization'] = process.env.VUE_APP_AUTH; // 공통 인증 코드
  instance.interceptors.request.use(
    config => {
      // 요청 성공 직전 호출됩니다.
      // config.headers['파라미터명'] = 값;
      return config;
    },
    error => {
      // 요청 에러 직전 호출됩니다.
      return Promise.reject(error);
    },
  );
  // 응답 인터셉터
  instance.interceptors.response.use(
    response => {
      let resData = response.data;
      //if (!isSuccess(resData)) {
      //    return Promise.reject({ status: 520, error: resData.error, message: resData.message });
      //}
      return resData;
    },
    error => {
      return Promise.reject(error);
    },
  );
  export default instance;
```

- interceptors에 의해서 요청과 응답 중간에 메시지 처리 가능. 
  - 요청시 인증에 대한 처리, 응답시 응답 결과에 대한 처리 등을 실행. 
  - 사용법.

  ```javascript
    import api from '@/src/api';

    ...

    let res = await api.get('/api/foo');
    console.log(res);
  ```

## 4. 프로파일 구성

- 최종 운영 배포할 경우는 빌드 과정을 통해서 웹팩으로 파일이 생성되는 과정이 있으며 이 과정을 통해 개발환경에 설정이 아닌 운영환경에 맞는 설정 적용이 필요. 
- vue cli에서는 기본 3개 모드.
  - development : serve 실행할 경우
  - production : build 실행할 경우
  - test : test 실행할 경우

- 각 프로파일은 "process.env.NODE_ENV"에서 선택된 프로파일을 식별가능.
- 프로젝트 루트 환경변수 파일 생성을 통해 각 프로파일 별 환경 설정
  - .env: 모든 프로파일에 적용됨
  - .env.local: 모든 프로파일에 적용됨(원격은 제외)
  - .env.프로파일명: 특정 프로파일에 적용 (예. 운영이라면 .env.production )
  - .env.프로파일명.local: 특정 프로파일에 적용(원격은 제외)
- local은 각자 본인 환경에 적용되는 환경설정 파일이기 때문에 .gitignore 추가 권장.
- 각 환경설정 파일에 환경설정변수 설정형식. 
  - VUE_APP_{변수명} = {값} 형식으로 설정 값을 부여.

  ```
    NODE_ENV = development
    VUE_APP_PORT = 8000
  ```

- NODE_ENV은 기본 프로파일들은 이미 제공되고 있기 때문에 별다른 설정은 필요없지만, 필요시 수정가능.
- 실제로 환경변수를 vue.config.js 파일에 적용한 예.

```javascript
  module.exports = {
    devServer: {
      port: process.env.VUE_APP_PORT || 8080
    }
  }
```

- 설정에 있는 값이 있으면 사용하고 없으면 기본으로 8080 포트를 사용하도록 구성.
- 쿠키 사용 : 쿠키를 사용하기 위해 vue-cookies 모듈 설치.

```
  yarn add vue-cookies
```

  - 사용법.

  ```
    import VueCookies from "vue-cookies";
    // 가져오기
    let foo = VueCookies.get('foo');
    // 저장하기
    VueCookies,set('foo', 'foo value', '10s');
  ```

### 기타 추가설정

#### npm repository 변경

- nexus를 사용한 repository 구성등 접속할 주소 변경이 필요할 경우 
- 프로젝트 루트에 .npmrc 파일을 생성하고 아래 내용을 추가한다.

```
  registry=https://foo.com/repository/npm-foo-proxy/
  _auth=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

- 인증에 관련된 추가 설정은 repository 서버 구성에 맞게 수정.

#### 빌드시 index.html 파일 이름 변경

- 빌드시 index.html 파일명이 아닌 다른 파일 명으로 생성된 빌드파일을 생성하도록 변경가능.
- vue.config.js 에 이하내용 추가.

```json
  module.exports = {
      ...
      indexPath: 'main.html',
  };
```

- index.html 파일이 아닌 main.html 파일로 빌드되도록 설정이 변경됨.

## Troubleshoot

### '<' 토큰 에러

- index.html 파일에 base 태그 및 기타 태그를 추가.

```html
  <head>
      <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
      <meta http-equiv="Pragma" content="no-cache">
      <meta http-equiv="Expires" content="0">
      <base href="/">
  </head>
```

- package.json 파일에 homepage 속성 추가.

```json
  { "homepage": "." …}
```

### 브라우저 캐시로 배포 문제
- 웹팩에 의해서 파일이 패키징되는 경우 파일명에 해시값을 붙여서 다른 파일로 배포.
- vue.config.js 파일에 아래 내용을 추가한다.

```javascript
module.exports = {
  ....
  configureWebpack: config => {
    if (process.env.NODE_ENV === 'production') {
        config.output.filename = 'js/[name].[chunkhash:8]].js';
    }
  },
}
```

- "chunkhash:8" 대신 "hash" 적용가능.

#### **webpack에서 제공하는 hash의 종류**
- hash, chunkhash, contenthash 
- 1. hash
  - hash를 사용할 경우 모든 output파일이 같은 hash 값으로 생성.
  - webpack 빌드시 고유한 hash 값 생성
- 2. chunkhash
  - chunkhash를 사용할 경우 모든 output파일이 chunk별로 chunkhash 값이 생성.
  - 파일 수정시 해당파일의 chunkhash 값만 변경.
  - 파일이 변경되지 않은 상태에서 빌드를 하게 되면 hash 값이 변경되지 않음.
- 3. contenthash
  - *ExtractTextPlugin으로 생성되는 css파일 같은 chunk에 포함되지않는 파일의 경우에는 [contenthash]를 사용.

#### 참고.1 Quasar pulgin 
- 개발 비용을 대폭 절감가능. 
- Vue.js 기반이며, 웹 사이트 및 앱에 대한 최신 UI (Material Guidelines를 따름) 사용가능.
- 데스크톱 및 모바일 브라우저 (iOS Safari 포함) 지원.
- 각 빌드 모드 (SPA, SSR, PWA, 모바일 앱, 데스크톱 앱 및 브라우저 확장)에 대한 동급 최고의 지원 및 자체 CLI와의 통합가능.
- 사용자 정의 (CSS) 및 확장(JS)이 쉽게가능.
- 성능에 가장 중점을 둔 프레임워크.
- 자동으로 tree-shakable 가능.


#### 참고.2 vue-cli + vuex + jwt 로그인
- vue-cli와 vuex 기반의 웹 애플리케이션에 JWT(Json Web Token)을 이용하여 로그인 기능 구현
- 로그인시 토큰 값을 쿠키(Cookies)에 저장
- 토큰 저장시 유효기간이 1분인 accessToken과 1시간인 refreshToken 이렇게 2개의 토큰을 저장
- 화면 전환, axios 요청을 할 때마다 토큰 값 체크
  - 토큰값 체크 시 accessToken만 없을 경우 refreshToken을 이용해 토큰재발급 및 저장
- 토큰 값 두 개가 모두 없을 경우 로그인 페이지로 이동

##### index.js

- src/router/index.js : 네비게이션 가드로 url이 변경될 때마다 토큰을 체크

```javascript
  //네비게이션 가드((뷰 라우터로 URL 접근에 대해서 처리할 수 있음)
  router.beforeEach( async(to, from, next) => { //여기서 모든 라우팅이 대기 상태가 됨
    /**
     * to: 이동할 url 정보가 담긴 라우터 객체
     * from: 현재 url 정보가 담긴 라우터 객체
     * next: to에서 지정한 url로 이동하기 위해 꼭 호출해야 하는 함수
     * next() 가 호출되기 전까지 화면 전환되지 않음
     */
    if(VueCookies.get('accessToken')===null && VueCookies.get('refreshToken') !== null){
      //refreshToken은 있고 accessToken이 없을 경우 토큰 재발급 요청
      await store.dispatch('refreshToken');
    }
    if (VueCookies.get('accessToken')){
      //accessToken이 있을 경우 진행
      return next();
    }
    if(VueCookies.get('accessToken')===null && VueCookies.get('refreshToken') === null){
      //2개 토큰이 모두 없을 경우 로그인페이지로
      return next({name: 'Login'});
    }
    return next();
  })
```

- 네비게이이션 가드 : router.vuejs.org/kr/guide/advanced/navigation-guards.html
  - vue-router에서는 주로 리디렉션하거나 취소하여 네비게이션을 보호하는 데 사용

##### App.vu

- src/App.vue : 컴포넌트가 변경될 때 마다 토큰을 체크

```javascript
  export default {
    name: 'App',
    created() {
      //메인 컴포넌트를 렌더링하면서 토큰체크
      let token = this.$store.getters.getToken;
      if (token.access == null && token.refresh == null) { //다 없으면 로그인 페이지로
        //이미 로그인 페이지가 떠있는 상태에서 새로 고침하면 중복 에러 떠서 이렇게 처리함
        this.$router.push({name: 'Login'}).catch(() => {}); 
      }
    },
    components: {
      'topMenu': menu
    }
  }
```

- 토큰 체크 : 토큰이 모두 없을 경우에는 로그인 창으로 이동.
- index.js에 네비게이션 가드 로직만으로는 url 변경이 아닌 컴포넌트만 변경되는 경우 대비불가.

##### login.js

- store/module/login.js

```javascript
import axios from "axios";
import VueCookies from 'vue-cookies';
//로그인 처리 관련 저장소 모듈
export const login = {
  state: {
      host: 'http://192.168.1.29:3000',
      accessToken: null,
      refreshToken: null
  },
  mutations: {
      loginToken (state, payload) {
          VueCookies.set('accessToken', payload.accessToken, '60s');
          VueCookies.set('refreshToken', payload.refreshToken, '1h');
          state.accessToken = payload.accessToken;
          state.refreshToken = payload.refreshToken;
      },
      refreshToken(state, payload) { //accessToken 재셋팅
        VueCookies.set('accessToken', payload.accessToken, '60s');
        VueCookies.set('refreshToken', payload.refreshToken, '1h');
        state.accessToken = payload;
      },
      removeToken () {
        VueCookies.remove('accessToken');
        VueCookies.remove('refreshToken');
      },
  },
  getters: {
    //쿠키에 저장된 토큰 가져오기
    getToken (state) {
      let ac = VueCookies.get('accessToken');
      let rf = VueCookies.get('refreshToken');
      return {        
        access: ac,
        refresh: rf
      };
    }
  },
  actions: {
    login: ({commit}, params) => {
      return new Promise((resove, reject) => {
        axios.post('/v1/auth/login', params).then(res => {
          commit('loginToken', res.data.auth_info);
          resove(res);
        })
        .catch(err => {
          console.log(err.message);
          reject(err.message);
        });
      })
    },
    refreshToken: ({commit}) => { // accessToken 재요청
      //accessToken 만료로 재발급 후 재요청시 비동기처리로는 제대로 처리가 안되서 promise로 처리함
      return new Promise((resolve, reject) => {
        axios.post('/v1/auth/certify').then(res => {
          commit('refreshToken', res.data.auth_info);
          resolve(res.data.auth_info);
        }).catch(err => {
          console.log('refreshToken error : ', err.config);
          reject(err.config.data);
        })
      })
    },
    logout: ({commit}) => { // 로그아웃
      commit('removeToken');
      location.reload();
    }
  }
}
```

- 토큰 관련 로그인, 토큰 재발급, 로그아웃, 토큰 반환 등 토큰 관련 로직이 필요한 파일들은 이 저장소를 사용.
- axios를 사용해서 토큰 재발급을 할 때 interceptors에서 재발급 요청을 한 후 처리하는데 비동기 통신이어서 무한으로 요청을 보내며 웹이 먹통이 되는 현상이 발생하여 actions에 promise를 사용.
  - actions 안에서 비동기 통신을 하는 부분은 promise로 처리 후 받는 쪽에선 async/await으로 처리추천.
- 위 소스에서 state의 토큰 부분과 mutation의 state에 토큰 값을 저장하는 건 사실상 사용안함.

##### axios.js

- src/service/axios.js : axios가 발생할 때마다 interceptors를 이용해서 토큰을 체크

```javascript
  //request 설정
  axios.interceptors.request.use(async function (config) { 
    if (config.retry==undefined) { //
      /**
       * axios 요청 중에 accessToken 만료시 재발급 후 다시 요청할 땐
       * 기존 요청 정보에서 retry=true만 주가되고 
       * 나머지는 그대로 다시 요청하기 때문에 url이 이상해져서 이렇게 나눔
       */
      config.url = store.state.login.host + config.url; //host 및 url 방식 수정필요
    }
    //헤더 셋팅
    config.timeout = 10000;
    config.headers['x-access-token'] = VueCookies.get('accessToken');
    config.headers['x-refresh-token'] = VueCookies.get('refreshToken');
    config.headers['Content-Type'] = 'application/json';
    // console.log(config);
    return config;
  }, function (error) {
    console.log('axios request error : ', error);
    return Promise.reject(error);
  });
  //response 설정
  axios.interceptors.response.use(
    function (response) {
      try {
        return response;
      } catch (err) {
        console.error('[axios.interceptors.response] response : ', err.message);
      }
    },
    async function (error) {
      try {
        //에러에 대한 response 정보
        const errorAPI = error.response.config; //요청했던 request 정보가 담겨있음
        //인증에러 및 재요청이 아닐 경우... (+재요청인데 refreshToken이 있을 경우)
        if (error.response.status == 401 && errorAPI.retry==undefined && VueCookies.get('refreshToken')!=null)  { 
          errorAPI.retry = true; //재요청이라고 추가 정보를 담음
          await store.dispatch('refreshToken'); //로그인 중간 저장소에 있는 토큰 재발급 action을 실행
          return await axios(errorAPI); //다시 axios 요청
        }
      } catch (err) {
        console.error('[axios.interceptors.response] error : ', err.message);
      }
      return Promise.reject(error);
  })
```

- axios가 발생할 때 accessToken이 없을 경우 다시 토큰을 재발급한 후 요청을 해야 할 때 status가 401이고 재요청한 axios가 아니고 refreshToken이 있을 경우에는, 다시 axios를 요청
  - 재요청 구분을 위해 retry = true를 사용, 토큰을 재발급한 후 다시 요청.
- 로그인 저장소의 actions안에 있는 refreshToken을 promise로 비동기를 처리하지 않으면 요청 중복현상이 발생하므로 주의
  
---
  
### 참고 1. vue https로 서비스
- nuxt.config.js

```json
  import * as fs from 'fs'

  server: {
    port: 3013,
    host: '0.0.0.0',
    https: {
      key: fs.readFileSync('/var/lib/jenkins/secrets/server.key'),
      cert: fs.readFileSync('/var/lib/jenkins/secrets/server.crt')
    }
  },
```

- package.json

```json
  "scripts": {
    "local": "nuxt --VUE_APP_ENV=local",
    "dev": "nuxt",
    "build": "nuxt build",
    "start": "nuxt start",
    "localbuild": "nuxt build --VUE_APP_ENV=local",
    "localstart": "nuxt start --VUE_APP_ENV=local",
    "devbuild": "nuxt build --VUE_APP_ENV=dev",
    "devstart": "export NODE_TLS_REJECT_UNAUTHORIZED=0 && nuxt start --VUE_APP_ENV=dev --env.NODE_TLS_REJECT_UNAUTHORIZED=0",
    "stgbuild": "nuxt build --VUE_APP_ENV=stg",
    "stgstart": "nuxt start --VUE_APP_ENV=stg",
    "prdbuild": "nuxt build --VUE_APP_ENV=prd",
    "prdstart": "nuxt start --VUE_APP_ENV=prd",
    "generate": "nuxt generate",
    "lint:js": "eslint --ext \".js,.vue\" --ignore-path .gitignore .",
    "lint": "npm run lint:js"
  },
```

- .env

```bash
# ##################################################
# 서비스 내부 정보
# ##################################################
# 서버 환경 정보 (local: 로컬테스트환경, ltd, dev,  prd)
VUE_APP_VERSION = version
VUE_APP_ENV = 'local'

# ##################################################
# 파일 업로드 용량 제한 (MB 단위)
# ##################################################
# 동영상
VUE_APP_MAXIMUM_FILE_SIZE_VIDEO = 50
# 이미지
VUE_APP_MAXIMUM_FILE_SIZE_IMAGE = 10
# 에디터
VUE_APP_MAXIMUM_FILE_SIZE_HTML = 30
# 기타파일
VUE_APP_MAXIMUM_FILE_SIZE = 50

# ##################################################
# API 서버경로
# ##################################################
#local
#개발자용
LOCAL_BASEURL = http://dev.xxx.com:8013
#dev
DEV_BASEURL = https://adm.xxxdev.com:8013

# ##################################################
# WEB 서버경로
# ##################################################
#local
# 개발자용
LOCAL_WEBBASE_URL = http://dev.xxx.com:3013
#dev
DEV_WEBBASE_URL = https://adm.xxxdev.com:3013

```

<details>
  <summary>Exp.</summary>  
  <pre>

  </pre>
</details>