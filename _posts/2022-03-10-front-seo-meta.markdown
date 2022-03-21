---
layout: default
title:  "9. SEO, META TAG"
date:   2022-03-08 22:41:00
categories: front
---

## **개요**
1. [SSR를 이용한 SEO 최적화](#1-ssr%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-seo-%EC%B5%9C%EC%A0%81%ED%99%94)
2. [META TAG 설정하기](#2-meta-tag-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0)

## **1. SSR를 이용한 SEO 최적화**
Next.js 를 사용하는 이유는 SEO 친화적인 웹사이트를 제작하기 위해서 입니다.

Next.js 에서는 **SSR, SSG**를 통한 **PreRendering**을 통해 SEO를 최적화 할 수 있습니다.

커뮤니티 게시글을 예시로 보여드리겠습니다.
<font color='coral'>SSR 적용 전</font>

![no_ssr_page](/assets/images/no_ssr_page.png)

게시글 데이터를 가져오기전 페이지가 로드되기때문에 생성된 html에서 게시글 내용이 존재 하지 않습니다.

<font color='coral'>SSR 적용</font>

```typescript
export const getServerSideProps: GetServerSideProps<IBoardSSRProps> = async (context) => {
    const id = context.params?.id;
    const response = await getBoardView(id as string);

    return {
        props: {
            board: response
        }
    }
}
```

`getServerSideProps()` 함수를 통해서 **SSR**을 사용 할 수 있습니다.

![ssr_page](/assets/images/ssr_page.png)

**SSR**를 적용하여 html이 **PreRendering** 되어 게시글 데이터가 모두 담아져있는 모습을 볼 수 있습니다.

## **2. META TAG 설정하기**

Next.js 에서는 `Next/Head` 컴포넌트를 이용하여 head태그를 조작 할 수 있습니다.

```html
<Head>
    <title>{board.title}</title>
    <meta name="description" content={board.content}/>
    <meta property="og:title" key="ogtitle" content={board.title}/>
    <meta property="og:description" key="ogdesc" content={board.content}/>
    <meta property="og:url" key="ogurl"
            content={`https://golfani.com/board/${boardQuery.data?.id}?type=${boardQuery.data?.boardType}&page=0`}/>
    <meta property="og:image" key="ogimage" content={board.urlList.length > 0 ? board.urlList[0] : 'https://golfani.com/og_img.png'}/>
</Head>
```

기본적으로 head 태그들을 사용하기 위해서 SSR을 적용하였습니다.

<font color='coral'>SSR 적용 전</font>

![no_ssr_meta_tag](/assets/images/no_ssr_meta_tag.png)

태그들이 정상적으로 동작하지 않는 모습을 보입니다.

<font color='coral'>SSR 적용</font>

![ssr_meta_tag](/assets/images/ssr_meta_tag.png)

태그들이 정상적으로 동작합니다!

### **OG(Open Graph) 태그**

OG 태그들을 이용하여 소셜네티워크상에서 페이지 정보를 미리 전달 할 수 있습니다.

![og_kakaotalk](/assets/images/og_kakaotalk.png)

```html
기본
<meta property="og:title" content="콘텐츠 제목" /> 
<meta property="og:url" content="웹페이지 URL" />
<meta property="og:type" content="웹페이지 타입(blog, website 등)" />
<meta property="og:image" content="표시되는 이미지" /> 
<meta property="og:title" content="웹사이트 이름" /> 
<meta property="og:description" content="웹페이지 설명" /> 

트위터 용도
<meta name="twitter:card" content="트위터 카드 타입(요약정보, 사진, 비디오)" /> 
<meta name="twitter:title" content="콘텐츠 제목" /> 
<meta name="twitter:description" content="웹페이지 설명" /> 
<meta name="twitter:image" content="표시되는 이미지 " /> 

모바일 앱 연결 시 노출되는 용도
<--iOS-->
<meta property="al:ios:url" content=" ios 앱 URL" />
<meta property="al:ios:app_store_id" content="ios 앱스토어 ID" /> 
<meta property="al:ios:app_name" content="ios 앱 이름" /> 
<--Android-->
<meta property="al:android:url" content="안드로이드 앱 URL" />
<meta property="al:android:app_name" content="안드로이드 앱 이름" />
<meta property="al:android:package" content="안드로이드 패키지 이름" /> 
<meta property="al:web:url" content="안드로이드 앱 URL" />
```

**Reference**

[https://velog.io/@byeol4001/Meta-Tag-OG%EC%98%A4%ED%94%88%EA%B7%B8%EB%9E%98%ED%94%84-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0](https://velog.io/@byeol4001/Meta-Tag-OG%EC%98%A4%ED%94%88%EA%B7%B8%EB%9E%98%ED%94%84-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0)

## **마침**

Next.js에서 제공하는 SSR뿐만 아니라 SSG을 사용하여도 SEO를 최적화 할 수 있습니다.

다만, SSG를 사용할때는 페이지의 데이터가 변경되지 않는 페이지를 대상으로 적용하여야 좋은 효과를 나타 낼 수 있습니다.
