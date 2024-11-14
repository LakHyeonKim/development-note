---
description: 마이그레이션 고민.. 사전 실험내용
---

# 📒 Webflux vs WevMVC (작성중)

## 테스트 환경

* core: 4
* mem: 1g

## 동영상 스트리밍 (range request)

### 사전 조건

* 요청 쓰레드 수: 20
* 요청 커낵션 수: 20
* 요청 시간: 60 s
* 요청 해더 (0 \~ 255KB)

```lua
 headers["Range"] = "bytes=0-261120"
```

### 결과













## 문서 10개 읽고 문서 하나 저장하기



<figure><img src="../.gitbook/assets/스크린샷 2024-10-24 오후 5.24.03.png" alt=""><figcaption><p>mvc</p></figcaption></figure>



<figure><img src="../.gitbook/assets/스크린샷 2024-10-24 오후 5.21.57.png" alt=""><figcaption><p>flux</p></figcaption></figure>
