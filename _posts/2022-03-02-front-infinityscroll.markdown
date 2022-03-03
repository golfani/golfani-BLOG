---
layout: default
title:  "5. 무한스크롤을 이용한 페이지네이션 (데이터 중첩현상 고려하기)"
date:   2022-03-01 16:40:00
categories: front
---

## **개요**
**GOLFANI**에서 제공하는 서비스중 `피드`서비스, `채팅`서비스 에 무한스크롤 페이지네이션 기능을 만들며 느낀 시행착오들을 정리한 글 입니다.

## **WHY 페이지네이션**?

**페이지네이션**이란 무엇일까요?

**페이지네이션**은 한 페이지에서 보여줄 데이터량이 많아서 사용자에게 효과적으로 제공할 수 없어
데이터를 페이지 별로 분리하여, 각 페이지마다 해당되는 데이터를 제공하는 방식입니다.

만약 페이지네이션을 사용하지 않는다면, 게시글이 1000개 존재할 시 한 페이지에서 모든 게시글이 보이게 되어
페이지 로드하는 시간이 상당히 길어지고, 사용자에게 불편함을 초래합니다.

적절한 양의 데이터를 사용자들에게 제공해 줘야 합니다.

## **무한스크롤을 이용한 페이지네이션**

기본적으로 페이지네이션은 우리가 흔히 사용하는 커뮤니티 사이트의 게시판에서 [이전 / 다음] 버튼으로 쉽게 확인할 수 있습니다.

피드 와 채팅 서비스에서는 다음 페이지의 데이터로 새롭게 변경되는 것이 아니라, 이전 데이터에 계속해서 새롭게 추가되는 기능이 필요합니다.  
또한, 저희가 원하는 페이지네이션 기능은 버튼으로 동작하는 페이지네이션이 아닌 스크롤을 감지하여, 스크롤이 일정 위치에 다다르게 되면 다음 페이지를 불러오는 기능이 필요했습니다.

**즉, 스크롤을 감지하여 원하는 위치에 도달했을 경우 다음 페이지의 데이터를 읽어와 이전 데이터에 추가해주는 기능이 필요합니다.**

### **스크롤 감지하기**

스크롤을 감지하는 방법으로 **Intersection Observer API**를 이용하였습니다.  
[참고문헌][observer-link]

스크롤을 감지하는 방법을 그림으로 나타내면 아래와 같습니다.

![scroll1](/assets/images/infinity_scroll_1.png)

1. 실제 데이터가 추가되는 영역  
2. 스크롤을 감지할 영역

Observer를 이용하여 스크롤이 감지하는 영역과 교차되는지 감지합니다.

```typescript
// 스크롤을 감지할 Div Element 지정
const scrollRef = useRef<HTMLDivElement>(null);
// observer Instance를 저장할 변수
const observer = useRef<IntersectionObserver>();

// Observer callback 함수 생성
const intersectionObserver = (entries : IntersectionObserverEntry[], io : IntersectionObserver) => {
    // 가시성 변화감지
    entries.forEach(async (entry)=> {
        // 교차 되었을경우
        if(entry.isIntersecting) {
            io.unobserve(entry.target);
            // 다음 페이지가 존재할경우 다음 페이지 데이터 불러오기
            hasNextPage && await fetchNextPage();
        }
    });
}

useEffect(() => {
    // Instance 생성하기
    observer.current = new IntersectionObserver(intersectionObserver);
    // 해당 요소 관찰 시작
    scrollRef.current && observer.current?.observe(scrollRef.current);
},[data]);
```

### **페이지네이션**

스크롤을 감지하였다면, 다음 페이지의 데이터를 불러와 봅시다.  
**React-Query** 의 **InfinityQuery**를 이용해 페이지네이션을 이용한 서버 상태 관리를 하였습니다.

처음에 다음 페이지를 불러올 때는 **(현재 페이지 + 1)에 해당하는 페이지**를 불러오게 로직을 만들었습니다.

그러다 어느 날 문득 생각났습니다.

    만약 A라는 사용자가 1번 페이지에서 2번 페이지를 부르고 있을 때  
    B라는 사용자가 새로운 컨텐츠(데이터)를 작성했다면?  
    2번 페이지에 1번 페이지의 마지막 데이터가 포함되어 있지 않을까?

**일반적인 상황**
![scroll2](/assets/images/infinity_scroll_2.png)

**중첩발생 상황**
![scroll3](/assets/images/infinity_scroll_3.png)

따라서 기존에 작성했던 로직을 변경하게 되었습니다.
기존 (현재 페이지 + 1) 에 해당하는 페이지를 불러오는 것이 아니라, **마지막 데이터의 id**를 통해 (**해당id + 1) 부터 데이터**를 가져오도록 하였습니다.

**InfinityQuery**에서 **PageParam**으로 다음 페이지를 넘겨주는 것이 아니라, **마지막 데이터의 id**를 넘겨 주도록 변경하였습니다.

새로운 데이터는 **(가장 최근 데이터 ~ 마지막 데이터 id + 페이지네이션 size)만큼** 데이터를 가져오게 됩니다.

```typescript
getNextPageParam : (lastPage) => {
    // 다음페이지가 없으면 undefined를 반환합니다.
    if(lastPage.totalPages <= 1) {
        return undefined;
    }
    // 다음페이지가 존재하면 가장 마자믹 데이터의 id를 반환합니다.
    return lastPage.content && lastPage.content[lastPage.content.length-1].id;
},
```

그리고, 페이지 초기 로딩 시  
초기 데이터를 가져올 때는 넘겨줄 id값이 없으므로, **가장 큰 정수형 값**을 넘겨줬습니다.  
데이터는 항상 최근 기준으로 가져오기 때문입니다!

이로써 피드, 채팅 서비스에서 사용되는 무한스크롤 페이지네이션 기능을 완성하였습니다.

무한스크롤로 페이지네이션을 구현할 때 데이터가 중첩되는 현상을 고려해 보세요!


[observer-link]: https://velog.io/@elrion018/%EC%8B%A4%EB%AC%B4%EC%97%90%EC%84%9C-%EB%8A%90%EB%82%80-%EC%A0%90%EC%9D%84-%EA%B3%81%EB%93%A4%EC%9D%B8-Intersection-Observer-API-%EC%A0%95%EB%A6%AC#intersection-observer-api%EB%9E%80