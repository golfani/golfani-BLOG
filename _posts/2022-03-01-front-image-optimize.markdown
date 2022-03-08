---
layout: default
title:  "4. 이미지 성능 최적화(Optimize)"
date:   2022-02-28 16:40:00
categories: front
---

## **목차**
1. [개요](#1-%EA%B0%9C%EC%9A%94)
2. [Next/Image](#2-nextimage-%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%9C-%EC%9D%B4%EB%AF%B8%EC%A7%80-%EC%B5%9C%EC%A0%81%ED%99%94optimize)
3. [LazyLoading](#3-lazyloading)

---

## **1. 개요**
앞선 포스팅을 통해서 어떻게 이미지들을 리사이징하고 여러 사이즈들의 이미지들을 관리하는지 알아보았습니다.

이번 포스팅은 사용자들에게 이미지들을 어떻게 빠르게 제공할 수 있을까?에 대한 포스팅입니다.

---

## **2. Next/Image 를 사용한 이미지 최적화(Optimize)**
Next.js 에서는 Next/Image 라는 모듈을 통해서 앞선 포스팅에서 다뤘던 이미지 라사이징과 최적화를 제공합니다.

### **예시**
```typescript
import Image from 'next/image'
import profilePic from '../public/me.png'

function Home() {
  return (
    <>
      <h1>My Homepage</h1>
      <Image
        src={profilePic}
        alt="Picture of the author"
        // width={500} automatically provided
        // height={500} automatically provided
        // blurDataURL="data:..." automatically provided
        // placeholder="blur" // Optional blur-up while loading
      />
      <p>Welcome to my homepage!</p>
    </>
  )
}
```
만약, 외부 이미지를 사용하고 싶으시다면 설정이 필요합니다.  
`next.config.js` 파일에서 이미지 URL을 요청하는 도메인을 추가하시면 됩니다.
```typescript
module.exports = {
  images: {
    domains: ['example.com', 'example2.com'],
  },
}
```
이미지 리사이징은 Image 컴포넌트의 width, height 를 입력하면 비율에 맞게 리사이징 됩니다.

또한, Next/Image 모듈로 로딩된 이미지는 next cache에 저장되어 동일한 이미지를 불러올 때  
새로 불러오는 것이 아니라 캐시된 데이터에서 가져와 이미지 로딩 시간을 매우 감소시켜 줍니다.

### **적용 예시**

<font color='coral'>기존 img 태그를 사용해서 이미지를 로드했을 경우 입니다.</font>

![normal_img](/assets/images/normal_img.png)
파일 확장자는 jpeg, 용량은 47.7kb, 로드 시간은 39ms 입니다.

<font color='coral'>Next/Image를 사용한 이미지를 로드했을 경우 입니다.</font>
![next_image](/assets/images/next_image.png)
파일 확장자는 webp, 용량은 230B, 로드 시간은 539ms 입니다.

<font color='coral'>다음은, 한번 로드된 Next/Imag를 다시 로드할 경우 입니다.</font>
![next_image_cache](/assets/images/next_imag_cache.png)
파일 확장자는 webp, 용량은 230B, 로드 시간은 3ms 입니다.

보신 거와 같이, 용량은 같지만 로드 시간이 확연하게 차이 날 수 있는걸 볼 수 있습니다.

하지만, 초기 로딩이 느리다는 단점이 존재하는데 이는 Next.js 공식 문서를 통해 해결할 수 있습니다.

> The next/image component's default loader uses squoosh because it is quick to install and suitable for a development 
environment.
When using next start in your production environment, it is strongly recommended that you install sharp by running yarn add sharp in your project directory.
This is not necessary for Vercel deployments, as sharp is installed   automatically.

바로 **sharp** 를 설치하면 된다는 설명입니다.

이러한 방식으로 Next/Image를 사용하여 이미지를 최적화 할 수 있습니다.

더 많은 정보는 [https://nextjs.org/docs/api-reference/next/image](https://nextjs.org/docs/api-reference/next/image) 에서 확인 하실 수 있습니다.

---

## **3. LazyLoading**

다음은 LazyLoading을 이용하여 성능 최적화 입니다.

### **LazyLoad?**

**LazyLoad** 란 페이지에서 해당 리소스가 실제로 필요할 때까지 해당 리소스의 로딩을 미루는 것입니다.  
즉, 페이지를 로드했을 때 모든 리소스를 로딩하는 방식이 아닌 필요할 때까지 로딩을 지연하는 것입니다.


이미지에 LazyLoading 방식을 적용하는 것입니다.  
그렇게 되면 페이지에 들어왔을 때 모든 이미지를 로드하는 것이 아니라, 해당 이미지가 보일때 로드하게 됩니다.

만약, 피드 서비스에서 5개의 사진이 있는 피드가 5개가 보이게 된다면 이미지는 총 25장입니다.  
페이지에 접속했을 때 25장의 사진을 한꺼번에 모두 로드하는 것보다 해당 피드가 보일 때 해당하는 이미지들만 로드하는 것이 훨씬 효율적입니다.

### **LazyLoadImage 컴포넌트 만들기**

어떻게 하면 LazyLoad 이미지를 구현 할 수 있을까요?

**GOLFANI**에서는 **IntersectionObserver API**를 이용해 구현하였습니다.  
아래는 구현 코드입니다.
```typescript
const LazyImage = ({src, className, id, onClick, onDoubleClick}: ILazyImageProps) => {
    const [isLoading, setIsLoading] = useState(false);
    const ref = useRef<HTMLDivElement>(null);
    const observer = useRef<IntersectionObserver>();

    useEffect(() => {
        observer.current = new IntersectionObserver(intersectionObserver);
        ref.current && observer.current?.observe(ref.current);
    }, []);

    const intersectionObserver = (entries: IntersectionObserverEntry[], io: IntersectionObserver) => {
        entries.forEach((entry) => {
            if (entry.isIntersecting) {
                io.unobserve(entry.target);
                setIsLoading(true);
            }
        })
    }

    return (
        <div ref={ref}>
            <img src={isLoading ? src : '로딩 이미지'} id={id} className={className} onClick={onClick}
                 onDoubleClick={onDoubleClick} alt={'lazy_img'}/>
        </div>
    )
}

export default LazyImage;
```

### **실제 적용**

![lazy_load](/assets/images/lazy_load.gif)

네트워크 탭을 보시게 되면, 이미지를 한 번에 불러오는 것이 아니라 필요할 때마다 이미지를 로드하는 것을 확인하실 수 있습니다.

---

## 마침

이미지는 현대 웹 사이트에서 빠질 수 없는 요소입니다.

이미지가 차지하고 있는 비중이 많이지는 추세인데 이미지 최적화를 통해 웹 성능 향상을 이끌어 낼 수 있습니다.
