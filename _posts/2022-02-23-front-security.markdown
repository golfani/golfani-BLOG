---
layout: default
title:  "2. Security 와 Token 관리"
date:   2022-02-22 20:00:00
categories: front
---

## **목차**
0. [인증방식 선택](#1-%EC%9D%B8%EC%A6%9D%EB%B0%A9%EC%8B%9D-%EC%84%A0%ED%83%9D)
1. [로그인 및 토큰 관리](#2-%EB%A1%9C%EA%B7%B8%EC%9D%B8-%EB%B0%8F-%ED%86%A0%ED%81%B0-%EA%B4%80%EB%A6%AC)
2. [Security 처리](#3-security-%EC%B2%98%EB%A6%AC)

## 1. 인증방식 선택
인터넷 상에서 데이터를 주고 받기 위해서 **HTTP** 프로토콜이라는 통신 규약을 따르게 됩니다.  

큰 부분으로 생각하면 클라이언트(요청) - 서버(응답)으로 나뉘게 됩니다.  

**HTTP**는 `비연결성(Connectionless)`, `무상태성(Stateless)` 특징을 가지고 있어 한 요청에 대한 응답이 끝날경우 연결은 끊어지게 됩니다. 또한 서버에서는 이전 클라이언트에 대한 정보 및 상태가 남아 있지 않습니다.

**HTTP**에서 보완을 하려고 한다면 클라이언트 - 서버 간 통신이 이루어질때 
각각의 요청마다 해당 요청이 유효한 요청인지 식별할수 있게 정보를 전달해야 합니다. 
이때 사용하는 기술이 `Cookie`, `Session`, `Token(JWT)` 방식입니다. 

### **Cookie**
**쿠키**는 클라이언트에 `key, value` 쌍으로 저장되는 데이터 입니다.  

클라이언트에서 서버에 요청을 보낼때 마다 쿠키에 저장되어있는 데이터들을 요청의 **헤더(Header)**에 담아 보내게 됩니다.  
서버에서는 헤더에 날라온 쿠키데이터를 기반으로 클라이언트를 **식별**할 수 있습니다.  

쿠키는 사용자의 브라우저에 저장되기 때문에 `XSS(Cross Site Scripting)`공격으로 데이터를 탈취 당할수 있습니다.

### **Session**
브라우저에 데이터를 저장하는 Cookie와 달리 서버에 데이터를 저장하는 방식인 **Session** 방식이 존재합니다.  

하지만 세션도 중간과정에서 탈취 당할 위험이 존재합니다.  

또한 서버에서 세션 저장소를 사용하게되어, 요청이 많아지게 되면 서버에 부하가 심해집니다.

### **토큰( JWT )**
쿠키와, 세션의 문제점들을 보완하기 위해서 **JWT**방식의 토큰이 등장하게 되었습니다.  

`JWT(JSON WEB TOKEN)`이란 필요한 정보들을 암호화한 토큰들을 의미합니다.  

토큰들은 요청 헤더에 담아 서버에서 토큰을 식별하게 됩니다.

JWT 는 3가지 구조를 가집니다.

    1. Header : alg(해싱 알고리즘), typ(토큰 타입)
    2. Payload : key - value 로 이루어진 데이터(Claim)
    3. Signature : 인코딩 한 Header + Payload 를 해쉬하여 생성한 값, 서버측에서 관리


**GOLFANI** 에서는 클라이언트를 식별하기 위한 방법으로 **JWT** 방식을 선택했습니다.  

JWT 방식이 단점이 없는 완벽한 방식은 아니지만 Cookie, Session 과 달리 탈취를 당하더라도
`무결성`을 보자하기 때문에 **위변조의 위험**에서 벗어날수 있습니다.

## 2. 로그인 및 토큰 관리

### **로그인 프로세스**
위 설명과 같이 GOLFANI 에서는 인증방식에서 JWT 방식을 채용했습니다.  
실제 유저가 로그인할 경우 일어나는 프로세스들을 보여드리도록 하겠습니다.

1) 클라이언트(React) 로그인 요청 -> 서버(Spring)  
2) Spring Security 에서 해당 요청이 유효한 사용자 ID, PW 인지 확인  
3-1) 유효한 사용자일 경우 -> **Response Header**에 `AccessToken`, `RefreshToken` 담아서 클라이언트에게 제공  
3-2) 유효한 사용자가 아닐 경우 -> HTTP STATUS 401 제공

#### **Redux Saga를 이용한 로그인 처리**
로그인 요청(비동기 요청) 관리를 **Redux-Saga**를 통해 관리 하였습니다.
```typescript
function* handleLoginSaga(action: PayloadAction<LoginMember>) {
    try {
        // 로그인 요청
        const user: IUser = yield call(login, action.payload);
        // 토큰 값 넣어주기
        yield securityAxios.defaults.headers.common['Authorization'] = `Bearer ${user.accessToken}`;
        // 쿠키 설정
        yield setCookie('userId', user.userId, {
            path: '/',
            secure: false,
            maxAge: 60 * 60 * 24 * 7,
        });
        // loginAsyncSuccess Action dispatch 해주기
        yield put(loginAsyncSuccess());
    } catch (error) {
        // 로그인 실패 에러 처리
        yield put(loginAsyncError({error: "아이디, 비밀번호가 일치하지 않습니다."}));
    }
}

export function* loginSaga() {
    // loginAsync 액션으로 날라오면 handleLoginSaga로 연결해줍니다.
    yield takeEvery(loginAsync, handleLoginSaga);
}
```
#### **AccessToken, RefreshToken**
기존 단일토큰 JWT 방식을 사용하게 되면 토큰이 제3자에게 탈취 당했을 경우, 토큰이 유효한 기간동안 보안에 취약하게 됩니다.  
-> 토큰의 유효시간을 길게 가져가면 보안에 더 취약하게 됩니다.  
만약, 토큰 유효기간을 짧게 가져간다면 사용자 입장에서는 로그인을 새롭게 계속해서 토큰을 새로 발급 받아야 하므로 불편함을 겪게 됩니다.

이때, 토큰의 유효기간을 짧게 가져가면서 사용자가 편리하게 느낄수 있는 방법이 없나? 에 대한 답이  
`AccessToken`, `RefreshToken` 두가지 토큰을 이용한 JWT 방식입니다.  

**AccessToken**

    토큰의 유효시간을 짧게 가져갑니다.
    실제 서버와 통신할때 인증에 사용되는 토큰입니다.

**RefreshToken**

    토큰의 유효기간을 길게 가져갑니다.
    AccessToken의 유효기간이 만료되었을 경우 RefreshToken을 이용해 AccessToken을 재발급 하게 됩니다.

**AccessToken**이 탈취 당하게 되면 보안에 취약한점은 동일하지만, 유효시간은 짧게 가져갈 수 있습니다.  
**RefreshToken**이 만료하게 되면 새롭게 로그인하여 토큰을 재발급 받습니다.  
또한 **RefreshToken**이 탈취 당하게 되면 긴 유효시간 동안 보안에 취약하기 때문에 **RefreshToken**를 잘 관리해야합니다.

### **토큰 관리**
**GOLFANI**에서는 어떻게 토큰을 안전하게 관리할까요?

로그인에 성공했을 경우 Response Header에 `AccessToken, RefreshToken`을 담아 옵니다.  

토큰관리에 가장 중요한 점은 토큰이 제3자에게 공개되지 않는 것입니다.  
`localstorage, sessionstorage, cookie`에 저장하게 되면 XSS 공격에 취약하게 되어 토큰을 탈취당할 가능성이 높습니다.  
실제로 인증에 사용하는 `AccessToken` 클라이언트에 저장하지 않습니다.  
따라서, Response로 담아온 `AccessToken`을 Axios Header에 바로 설정하게 됩니다.

**RefreshToken**은 Axios **Header**에 담아 놓고 있으면, 매 요청마다 헤더에 토큰이 담아져있어  
유효시간이 상대적으로 긴 **RefreshToken**은 적절하지 않습니다.

그렇다면 어디에 저장하고 있어야 할까요?
`HTTP Only + secure Cookie`
HTTP Only 설정이 된 쿠키는 값을 접근 할 수 없습니다.
secure 설정이 된 쿠키는 https 에서만 동작하게 됩니다.

AccessToken이 만료 되었을때, RefreshToken만 쿠키에 담아서 서버에 전달 해주게 됩니다.

## 3. Security 처리

### **초기 Security 관리**
**GOLFANI** 초기에는 모든요청에 대해서 같은 axios를 통해 서버 통신을 진행했습니다.
토큰이 필요하지 않은곳에도 항상 토큰을 담아주게 되어 불필요한 요청이 날라가게 되었습니다.

또, AccessToken이 만료되게 되면 axios **interceptor**를 통해서 토큰을 재발급 받는 방식으로 진행했습니다.
#### **Axios Interceptor?**
Axios Interceptor란 Axios가 then, catch로 처리 되기전에 요청이나 응답을 가로 챌 수 있습니다.

아래는 인터셉터를 활용하여 특정요청이 401에러를 만났을경우 토큰을 재발급하는 로직입니다.

![interceptor](/assets/images/axios_interceptor.png)

**토큰 재발급 과정**

    1) 만료된 토큰으로 API요청
    2) 권한 없음으로 요청 실패
    3) 토큰 재발급요청
    4) 기존 요청 재요청

이러한 과정은 유저 입장에서는 딜레이가 발생한다고 느낄수 있을거 같았습니다.  
이는 저희가 원하는 프로세스 방향이 아니였습니다.

### **현재 GOLFANI**
#### **요청 분리**
**클라이언트 - 서버**간 요청을 할때 `인증이 필요하지 않은 요청`, `필요한 요청`으로 분리 할 수 있습니다.  
예를 들면 피드페이지에서 로그인을 하지 않은 유저도 피드 게시글들을 확인할 수 있습니다. 하지만, 피드 작성은 할수 없습니다.  


인증이 필요하지 않은 요청은 기존 axios 요청을 이용하여 통신을 하게 되고
인증이 필요한 요청은 AccessToken이 Header에 담긴 axios 를 이용해 통신을 진행합니다.
**GOLFANI**에서는 AccessToken이 Header에 담긴 axios 를 securtyAxios로 사용합니다.

#### **AccessToken 재발급**
**GOLFANI** 에서는 엑세스 토큰이 **만료되기전**에 재발급 요청을 하는 프로세스를 선택했습니다.  
따라서 위와 같은 딜레이 문제를 유저가 느끼지 못하게 할 수 있습니다.

토큰 재발급 로직은, 실제 유저가 눈치채지 못하게 조용히 진행됩니다.  
엑세스 토큰의 만료시간이 15분일 경우,  
만료 되기 5분전에 토큰을 재발급 하게 됩니다.

```typescript
const onRegenerateAccessToken = async (userId: string) => {
    try {
        const response = await regenerateAccessToken(userId);
        // Token이 필요한 곳에 헤더로 넣어줍니다.
        securityAxios.defaults.headers.common['Authorization'] = `Bearer ${response.data}`;
        socket.socketClient.connectHeaders['Authorization'] = response.data;
    } catch (e) {
        // 토큰 재발급 실패 시
        removeCookie('userId');
        window.location.href = 'login';
        alert('로그인 세션 만료, 다시 로그인 해주세요');
    }
}

export const onSilentRefresh = async (userId: string) => {
    const ACCESS_TOKEN_EXPIRE = 1000 * 60 * 15; // ms
    await onRegenerateAccessToken(userId);
    setTimeout(async () => {
        await onSilentRefresh(userId);
    }, ACCESS_TOKEN_EXPIRE - 6000 * 5); // ms
}
```

페이지가 새롭게 로드 될때마다 Refresh 진행됩니다.

```typescript
// _app.tsx
useEffect(() => {
    if(로그인) {
        // silentRefresh 진행
        onSilentRefresh(userId);
    }
}, []);
```
이러한 방식으로 엑세스 토큰을 재발급하는 프로세스를 만들었습니다.

## **마침**
**GOLFANI** 에서 로그인, 토큰관리를 통해 클라이언트 - 서버 간 인증 방식들과 토큰 저장방식에 따른 보안문제도 이해 하게 되었습니다. 
