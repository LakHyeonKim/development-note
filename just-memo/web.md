# 📒 Web 정적데이터 캐시 하는 방법

## 특정 endPoint에 캐싱 해더 추가 (Cache-Control)



```java
// Some code

@Component
public class CachingFilter extends HttpFilter {

  @Value("${builder.static-resource-cache-age:86400}")
  private long staticResourceCacheAge;

  @Override
  protected void doFilter(HttpServletRequest request, HttpServletResponse response,
    FilterChain chain)
    throws IOException, ServletException {

    String uri = request.getRequestURI();

    if (uri.startsWith("/api/file")) {
      response.setHeader("Cache-Control",
        String.format("max-age=%s", Math.max(staticResourceCacheAge, 0)));
    }

    chain.doFilter(request, response);
  }
}
```



## 크로미움 브라우저 Https self signed certificate cache issue

Chromium 기반 브라우저의 알려진 문제 HTTPS 연결이 잘못된 인증서를 사용하는 경우 응답을 캐시하지 않음

#### 이슈 관련 링크

{% embed url="https://github.com/MicrosoftEdge/WebView2Feedback/issues/2634" %}

{% embed url="https://issues.chromium.org/issues/40666473" %}

### 파이어포스 vs 크롬

cache-control 적용 상태이고 self signed scertificate 인 htttps를 적용 하였을 때

#### 크롬

<figure><img src="../.gitbook/assets/병신크롬.gif" alt=""><figcaption></figcaption></figure>

#### 파이어폭스

<figure><img src="../.gitbook/assets/파이어폭스.gif" alt=""><figcaption></figcaption></figure>



