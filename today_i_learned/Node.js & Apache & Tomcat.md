### 노드, 아파치, 톰캣은 모두 리소스를 저장해 놓는 server

1. Apache (Static pages) + php (Dynamic pages - PHP to HTML)
- php는 **Web(App) server**
- asdf.php -> Apache에서 php파일 접근 도와줌 -> php엔진이 asdf.php 해석 후 HTML로 변환 -> Apache에 static page로 HTML 문서를 넘김 -> 사용자(browser)에게 serve

2. Apache (Static pages) + Tomcat (Dynamic pages - JSP to HTML)
- Apache는 **HTTP server** : HTML 문서를 Browser에 Serve한다(띄워준다)는 의미
- Tomcat은 **Web(App) server** : convert dynamic pages to static(HTML) pages -> send to Apache
- asdf.jsp -> Tomcat에서 접근 + 해석 후 HTML로 변환 -> Apache에 static page로 바뀐 HTML 문서를 넘김 -> 사용자(browser)에게 serve
- asdf.html -> Apache에서 접근 -> 사용자(browser)에게 serve

3. Node - Javascript Interpretation (Dynamic pages - Javascript to HTML) + HTML server 두 개의 역할을 모두 다 함!
- HTTP 서버도 포함되어 있기 때문에 Apache/Nginx와 같은 HTTP 서버가 따로 필요 X
- 여기서 Web(App) server는 DB 접근 + 해당 데이터로 HTML 페이지 생성 담당
- express.js (Web(App) server - Tomcat과 유사) : Node에 기능 추가한 라이브러리

4. Node란?
- Javascript의 런타임 환경이다. 본래 브라우저에 내장되어 있던 Javascript 엔진(Chrome의 V8)을 브라우저 밖으로 떼어내어 외부 장소에서도 Javascript를 돌릴 수 있는 환경을 새롭게 조성한 것이다.
- Node를 사용하는 이유: 서버를 구축하기 쉬우며 논블로킹 I/O 방식을 채택하기 때문.
