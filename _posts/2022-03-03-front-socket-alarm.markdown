---
layout: default
title:  "6. 소켓(STOMP)를 이용한 실시간 알람 및 채팅 서비스"
date:   2022-03-02 15:24:00
categories: front
---

**GOLFANI**에서 WebSocket을 이용해 실시간 알람 서비스 및 채팅 서비스를 개발하면서 겪은 **트러블슈팅**과 **각종 이슈**들을 정리한 글 입니다.

## 목차
1. [Why WebSocket & STOMP](#1-why-websocket--stomp)
2. [소켓 기본 설정](#2-%EC%86%8C%EC%BC%93-%EA%B8%B0%EB%B3%B8-%EC%84%A4%EC%A0%95)
3. [실시간 알람 서비스](#3-%EC%8B%A4%EC%8B%9C%EA%B0%84-%EC%95%8C%EB%9E%8C-%EC%84%9C%EB%B9%84%EC%8A%A4)
4. [채팅 서비스](#4-%EC%B1%84%ED%8C%85-%EC%84%9C%EB%B9%84%EC%8A%A4)

## **1. Why WebSocket & STOMP**

### **WebSokcet**
사용자에게 피드, 게시글에 대한 반응(댓글, 좋아요)을 알람 서비스로 제공해 주고 싶었습니다.  
이때, 알람을 실시간으로 사용자에게 제공하여 사용성을 올려주고 싶었습니다.

또한, 실시간 채팅 서비스를 제공하려고 했긴 때문에  
실시간 통신에 적합한 **WebSocket**을 선택하게 되었습니다.

기존 HTTP통신 방식을 사용하게 되면 클라이언트 -> 서버로 단방향 통신만 가능하고  
클라이언트 <-----> 서버 양방향 통신은 불가능합니다.

**Socket**방식 통신은 클라이언트와 서버 간의 연결이 유지되어 있고 양방향 통신이 가능합니다.
![httpvssocket](/assets/images/http_vs_websocket.png)

### **STOMP**
**STOMP (Simple Text Oriented Messaging Protocol)**은 메세지 전송을 효율적으로 처리하는 프로토콜입니다.

PUB / SUB 구조로 되어있어, 메세지를 전송하고 메세지를 받아서 처리하는 부분이 확실하게 정해져 있습니다.

STOMP는 기본적으로 WebSocket위에서 동작하는 프로토콜입니다.  
클라이언트, 서버가 전달할 메세지의 유형, 형식, 내용들을 정의하는 매커니즘 입니다.

기본적인 구조로 `TOPIC`, `PUB`, `SUB`으로 구성되 어있는데 채팅 서비스로 예를 들면

    TOPIC : 채팅방  
    채팅방 입장 : 해당 TOPIC 구독  
    PUB : 메세지를 보내는 행위  
    SUB : 메세지를 받는 행위

이런식으로 표현할 수 있습니다.

채팅서비스에서 주고받는 채팅 메시지, 알람 서비스에서 주고 받는 알람 메시지를 효과적으로 관리하기 위해
STOMP 프로토콜을 선택했습니다.

또한 **STOMP**는 메세지 **Header**를 사용할 수 있어, **GOLFANI에서 사용하는 Security**를 소켓에도 그대로 적용 할 수 있습니다.

## **2. 소켓 기본 설정**
알람 서비스는 상시적으로 동작하기 때문에, 항시 클라이언트는 소켓 서버에 연결되어 있어야 합니다.

따라서, 알람 서비스가 비정상적으로 동작하는 것을 방지하기 위해서 소켓 서버에 정상적으로 연결되어 있을 경우에만 페이지를 렌더링 해주는 방식으로 구현했습니다.

새로운 페이지를 진입하거나, 페이지를 나갔을 때 소켓 연결에 대한 로직이 이루어집니다.
```typescript
// _app.tsx
useEffect(() => {
    if(로그인) {
        // 로그인 상태일시 소켓연결 실행
        // 소켓연결에 성공하게 되면 실행하게될 callback함수들을 인자로 넘겨줍니다.
        socketConnect(alarmCallback,onSetSocketConnect,subForActivatedChat);
        }
    }

    return () => socketDisconnect();
}, []);

```
소켓 연결 함수 입니다.
```typescript
export const socketConnect = (callback : (data : IMessage) => void, listener : (state : boolean) => void, subChat : () => void) => {
    // 소켓 연결이 성공하면 실행됩니다.
    socket.socketClient.onConnect = async () => {
        // 알람 Topic Channel을 구독합니다.
        await subNoticeChannel(callback);
        // 추후 설명
        // 현재 입장한 채팅방의 채팅 Topic을 구독합니다.
        await subChat();
        // 콜백함수 실행
        await listener(true);
    }
    // 연결 실패했을경우 실행됩니다.
    socket.socketClient.onDisconnect = () => {
        listener(false);
    }
    // 소켓 연결을 시작합니다.
    socket.socketClient.activate();
}
```
## **3. 실시간 알람 서비스**

먼저, 알람 서비스에서 소켓통신을 사용하기 위해서 유저는 **자신의 유저ID**를 기반으로 하는 **TOPIC을 구독**하게 됩니다.  
유저ID의 topic을 이용해 소켓통신이 이루어지게 됩니다.

```typescript
const subNoticeChannel = (callback : (data : IMessage) => void) => {
    // 알람 채널 구독하기
    socket.socketClient.subscribe(`/queue/${userId}`,callback,{ id : noticeSubId, type : 'ALARM', userId : userId});
}
```

알람 서비스에서 소켓통신이 일어나는 경우는 2가지 입니다.

### **1) 다른 유저의 컨텐츠에 활동(댓글, 좋아요) 를 했을경우**

소켓을 통해 알람이 전송되는 과정을 살펴봅시다.

예를 들어, A유저가 B유저의 댓글에 좋아요를 눌렀을 경우입니다.
1. 댓글에 좋아요를 누른 액션(좋아요 누르기, 좋아요 취소) 에 대한 통신을 진행합니다.  
2. 해당 액션이 성공적으로 이루어지게 되면 소켓을 통해 알람을 전송합니다.

이때 고려해야 할 사항이 바로, 자기 자신에 대한 컨텐츠는 알람을 전송하지 않는 것입니다.  
자기 자신에게 알람을 보내게 되면 불필요한 알람이 쌓이게 됩니다.

저희는 userIsReplyLikesQuery라는 변수를 통해, 해당 컨텐츠가 자신의 컨텐츠인지 확인할 수 있습니다.

아래는 **GOLFANI**에서 사용하는 예시 코드입니다.

**<font color='coral'>댓글 좋아요에 대한 알람 보내기</font>**
```typescript
const onRegisterLikes = async () => {
    try {
        // 좋아요 액션에 대한 통신 진행
        const response = await registerLikesMutate.mutateAsync();
        // 통신이 성공적으로 이루어지면 소켓을 통한 알람 전송
        try {
            // 자기 자신컨텐츠에 대한 알람은 전송X
            userIsReplyLikesQuery.data || sendAlarmBySocket('LIKES', reply.userId, '댓글을 좋아합니다. ', reply.postId, reply.payload, 'POST_REPLY', reply.id);
        } catch (e) {
            // catch
        }
    } catch (e) {
        // catch
    } finally {
        // finally
    }
};
```

**<font color='coral'>알람 채널에 PUBLISH 해주기</font>**
```typescript
const publishAlarm = (payload : TAlarmSendDto) => {
    // 소켓이 연결 안됬을 경우 방지
    if (socket.socketClient.active && socket.socketClient.connected) {
        // pub 해주기
        const publish = socket.socketClient.publish({
            destination: `/alarm/${payload.receiver}`,
            body: JSON.stringify(payload),
        });
    }
    else {
        alert('socketClient not Connected');
    }
}
```

이러한 과정을 통해 알람이 전송됩니다.

### **2) 다른 유저가 나의 컨텐츠에 활동(댓글, 좋아요) 를 했을경우**
위 1)번 과정이 이루어지고 나면 구독하고 있던 알람채널에서 메세지가 도착하게 됩니다.

메세지가 도착하게 되면, 알람채널을 구독할때 등록한 **callback 함수**가 실행되게 됩니다.

**callback함수**를 통해서 유저에게 알람을 보여줍니다.

**<font color='coral'>callback함수는 현재 나의 알람 목록에 대한 쿼리를 무효화하고 새로운 알람데이터를 받아옵니다.</font>**

```typescript
const alarmCallback = async (data : IMessage) => {
    const message = JSON.parse(data.body);
    // 알람 타입 분기
    if(message.type === 'CHAT') {
        await queryClient.invalidateQueries('chatRoom');
        await queryClient.invalidateQueries('unReadMessage');
    }
    else {
        await queryClient.invalidateQueries('alarm');
        await queryClient.invalidateQueries('unReadAlarm');
    }
}
```

알람 서비스 과정에 대해 세세하게 모든 과정들을 보여드리진 않았지만, 전체적인 과정은 이렇게 진행됩니다.

## **4. 채팅 서비스**

채팅 서비스는 다른 사용자와 메세지를 실시간으로 주고받을 수 있는 서비스입니다.  
소켓을 통해 메세지를 주고받는 로직은 크게 다를 바 없습니다.

### **전체적인 과정**
채팅 서비스에서 이루어지는 전체적인 과정은 이러합니다.

1) 사용자가 채팅을 하기 위해 채팅방에 입장합니다 -> 해당 채팅방ID를 기반으로 TOPIC 구독  

2) 사용자가 채팅메세지를 보냅니다 -> 소켓을 통해 해당 메세지를 PUBLISH  

3) 상대방은 소켓을 통해 메세지를 확인합니다 -> SUBSCRIBE에 등록한 callback함수를 통해 메세지 업데이트

4) 채팅방을 나가게 되면 구독한 채팅방을 구독 해제 합니다.

채팅방에 입장하게 되면 구독하는 예시 코드 입니다.
```typescript
const subChatChannel = (roomId : number, callback : () => void) => {
    // callback함수를 받아 현재 구독중인 채팅방이 없을경우에만 해당 채팅방을 구독합니다.
    if (socket.chatRoomId === undefined) {
        socket.chatRoomId = roomId;
        const subId = `chat-sub-${roomId}`;
        const subscription = socket.socketClient.subscribe(`/topic/${roomId}`, callback, {
            id: subId,
            roomId: roomId.toString(),
            userId: userId,
            type: 'CHAT'
        });
    }
}
```

채팅 서비스에서 소켓을 이용한 통신은 크게 2가지로 나눌 수 있습니다.

### **1) 상대방이 채팅방에 존재하지 않을 경우**
상대방이 채팅방에 존재하지 않을 경우에는 위에서 사용한 **알람 서비스**를 이용하여 채팅 메세지를 상대방에게 전달하게 됩니다.

**알람 서비스**에서 등록한 **callback함수**를 통해  
나에게 도착한 읽지 않은 채팅 메세지 개수를 업데이트해줍니다.  
또한, 채팅 페이지에서 보여지는 각 채팅방별 가장 마자막 메시지도 업데이트해주게 됩니다.

### **2) 상대방이 채팅방에 존재할 경우**

상대방이 채팅방에 존재하는 경우에는 해당 채팅방ID에 메세지를 publish 하면, 해당 TOPIC을 subscribe 하고 있는 상대방이 존재하게 됩니다.

상대방은 채팅방을 구독할때 등록한 **callback함수**를 통해 채팅 메세지를 업데이트 하게 됩니다.

### **채팅방에 상대방이 들어와있는지 파악하는 방법**

어떻게 하면 해당 채팅방에 상대방이 들어와있는지 파악할 수 있을까요?

저희가 사용한 방법은 채팅방ID별 현재 참가한 사용자들을 저장하는 방법입니다.

맨처음, 사용자가 채팅방을 들어가 해당 채팅방을 구독하게 되면 서버에 소켓으로 채팅방을 참가한다는 정보를 제공합니다.  
```typescript
socket.socketClient.publish({
    destination: `/chat/subscribe/${roomId}`,
    body: 'participate',
});
```
서버에서는 채팅방ID, 사용자 정보를 통해서 `partcipate 해시 맵 변수`를 만들게 됩니다.

그 후, 채팅 메세지를 보낼 때 해당하는 채팅방ID를 통해 현재 참가중인 사용자들을 확인합니다.  
자신을 제외한 다른 사용자가 존재하면, 상대방이 채팅방에 들어왔다고 판단합니다.

채팅방을 나가게 되면 해당 해시 맵 변수에서 해당 사용자를 제거합니다.

![participate](/assets/images/participate.png)

## **마치며**

실제 서비스를 구현할 때는 처음 사용하는 웹소켓과, STOMP때문에 이런저런 자료를 찾아보고  
많은 시행착오를 겪으며 힘들게 완성했지만, 실제로 잘 동작하는 서비스를 보면 큰 성취감이 생깁니다.

