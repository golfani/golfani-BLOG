---
layout: default
title:  "8. 리액트 성능 최적화"
date:   2022-03-08 22:34:00
categories: front
---

## 목차
1. [개요](#1-%EA%B0%9C%EC%9A%94)
2. [Reconcilation(재조정) 피하기](#2-reconcilation%EC%9E%AC%EC%A1%B0%EC%A0%95-%ED%94%BC%ED%95%98%EA%B8%B0)
3. [Virtualize Long Lists (긴 목록 가상화)](#3-virtualize-long-lists-%EA%B8%B4-%EB%AA%A9%EB%A1%9D-%EA%B0%80%EC%83%81%ED%99%94)
4. [Dynamic Import](#4-dynamic-import)

## **1. 개요**

웹 어플리케이션의 사이즈가 커짐에 따라 최적화의 필요성을 느껴 최적화를 진행하였습니다.

리액트에서 성능 최적화를 위한 여러 가지 방법을 제공합니다.

가장 기본적인 방법으로는 프로덕션 빌드를 이용하는 것입니다.

React에는 유용한 경고가 많이 포함되어 있습니다.  
이 경고들은 개발하는 데 있어 매우 유용합니다.  
그러나 그 경고는 React를 더 크고 느리게 만들기 때문에 앱을 배포할 때 프로덕션 버전을 사용해야 합니다.

그리고 몇가지 속도를 높여주는 방법이 더 있습니다.

## **2. Reconcilation(재조정) 피하기**

컴포넌트의 prop이나 state가 변경되면 React는 새로 반환된 엘리먼트를 이전에 렌더링 된 엘리먼트와 비교해서 실제 DOM 업데이트가 필요한지 여부를 결정합니다.
같지 않을 경우 React는 DOM을 업데이트합니다.

React가 변경된 DOM 노드만 업데이트하더라도 리렌더링에는 여전히 다소 시간이 걸립니다.

이 문제를 피하는 가장 간단한 방법은 props와 state로 사용 중인 값의 변경을 피하는 것입니다.

### **React.memo**

**React.memo** 는 컴포넌트의 **props를 비교하여 리렌더링**을 막을 수 있는 **메모이제이션** 기법을 제공하는 함수입니다.

예를들어 피드페이지의 컴포넌트 구조는 다음과 같습니다.

    Index
        - FeedMain
            - FeedList
                - FeedItem

무한스크롤을 통한 페이지네이션으로 FeedList의 길이가 변할때,  
새로 변경된 FeedItem만 렌더링하고 기존에 이미 렌더링된 FeedItem들은 리렌더링 되지 않도록 만들어봅니다.

**<font color='coral'>React.memo 미 적용시</font>**

![no_memo](/assets/images/no_memo.gif)

기존 FeedItem도 리렌더링 되는 모습을 볼 수 있습니다.

**<font color='coral'>React.memo 적용시</font>**

![memo](/assets/images/memo.gif)

새롭게 추가되는 FeedItem만 리렌더링 됩니다.

```typescript
const FeedItem = ({feed, isModal} : IFeedItemProps) : JSX.Element => {
    return (
        // 렌더링
    );
};

export default React.memo(FeedItem);
```

컴포넌트에 React.memo를 감싸주는 것만으로
손쉽게 React.memo를 적용할 수 있습니다.

### **useCallback**

**useCallback**을 사용하면 특정함수를 새로 만들지 않고 재사용하고 싶을때 사용합니다.

```typescript
// useCallback 적용 전
const onRemove = (id) => {
    setUsers(users.filter(user => user.id !== id));
};

// useCallback 적용 후
const onRemove = useCallback((id) => {
    setUsers(users.filter(user => user.id !== id));
}, [users]);

// 함수형 업데이트 적용
const onRemove = useCallback((id) => {
    setUsers(users => users.filter(user => user.id !== id));
}, []);
```

`deps`에 `users`가 들어있기 때문에 `users`배열이 변경될때마다 함수가 새로 만들어집니다.

이때, 이걸 최적화 하고 싶으면 `deps`에서 `users`를 지우고 함수들에서 `useState`로 관리하는 `users`를 참조하지 않게 하는 것 입니다.

함수형 업데이트를 통해서 setUsers에 등록하는 콜백함수의 파라미터에서 최신 `users`를 참조 할 수 있기 때문에 deps에 users를 넣지 않아도 됩니다.


## **3. Virtualize Long Lists (긴 목록 가상화)**

만약 어플리케이션이 많은 데이터를 나열한다면(백개 혹은 천개의 행), "windowing" 이라고 알려져있는 기술을 사용하는 것을 추천한다고 합니다.

해당 기술은 특정 시점에 전체 row 중 일부분만 랜더링합니다. 즉 보이는 부분만 렌더링하는거죠.  
일종의 lazy-rendering 라고 볼 수 있겠네요. 이러한 기술은 컴포넌트를 리렌더링하는 시간을 극적으로 줄일 수 있습니다.

## **4. Dynamic Import**

Next.js에서 **Dynamic Import**를 지원하여 모듈을 빌드타임이 아닌 런타임에 로드 할 수 있도록 해줄 수 있습니다.  
이를통해 번들 파일을 분리하고 **퍼포먼스 향상**을 기대 할 수 있습니다.

`dynamic` 모듈을 이용하여 쉽게 작성할 수 있습니다.

**<font color='coral'>Dynamic Import 적용 전</font>**

![dynamic_before](/assets/images/dynamic_before.png)

**<font color='coral'>Dynamic Import 적용 후</font>**

```typescript
import dynamic from "next/dynamic";

const ShopStatisticPage = dynamic(() => import("src/components/shop/manage/statistic/ShopStatisticPage"));

const ShopStatistic = (): JSX.Element => {
    return (
        <div>
            <ShopStatisticPage/>
        </div>
    )
}

export default ShopStatistic;
```

![dynamic_after](/assets/images/dynamic_after.png)

해당 페이지들의 첫 JS로드 용량이 줄어든 것을 확인할 수 있습니다.

**Dynamic Import**를 통해 첫 로드 시간를 줄이는 방법을 알아보았습니다.  
앱의 페이지 개수가 늘어나고 관리하는 모듈, 번들의 개수가 증가하게 되면 효과가 더 커질 수 있습니다. 

## **마침**

앱의 성능은 사용자들에게 쾌적한 사용자 경험을 느낄 수 있도록 도와주기 때문에 가장 중요하고 항상 고려되는 부분입니다.  
앱의 규모가 커질수록 성능이 떨어질 수밖에 없는데 여러 가지 최적화를 통해 성능을 개선하려고 노력하여 좋은 사용자 경험을 느낄 수 있도록 노력해 봅시다!
