개발하는 와중에 겁나 어이없는 사건을 발견했다. 
도대체간에 tag를 `<include>`를 쓰면 안되고 `<merge>`를 쓰면 동작하는 현상이 왜 일어났는지 도통 알 수가 없다.
디자인 요구사항에 따르면 쿠팡이츠처럼

```
<AppBarLayout>
  <CollapseToolbarLayout>
    <Custom Layout>
    <Toolbar>

<TabLayout>
<ViewPager>
```
  
구조로 이루어진 화면이다.
그런데 Custom Layout을 `<include>`로 삽입할 경우 도무지 CollpaseToolbarLayout의 `app:contentScrim`이 먹질 않는 것이다.
그래서 이 요구사항과 정말 비슷하게 구현된 쿠팡이츠를 디컴파일해서 소스를 까봤다.
`<merge>`로 바꾼 결과 `app:contentScrim`이 멀쩡하게 동작하였다.
이런 시발???
CollapseToolbarLayout의 소스를 까봐도 구현사항을 전혀 못찾겠는데
이쯤 되면 내가 기본 중에 기본을 모른다는 소리 아닌가?

# 이유를 알았어!!
코드가 caching돼서 그런거엿음
