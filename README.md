# gildongE — 실시간 알림 확장

**차량 어시스턴트 백엔드에 SSE 기반 실시간 알림을 얹은 확장본**

[![Base](https://img.shields.io/badge/base-gildongE-6DB33F?logo=springboot&logoColor=white)](https://github.com/jgjoe/gildonge-backend)
[![Push](https://img.shields.io/badge/push-Server--Sent%20Events-blue)](#설계-판단)
[![Stack](https://img.shields.io/badge/Spring%20Boot-Java%2017-6DB33F?logo=springboot&logoColor=white)](#기술-스택)

[**gildongE**](https://github.com/jgjoe/gildonge-backend)(6인 팀 AI 차량 어시스턴트의 백엔드) 위에
**서버가 먼저 사용자에게 알림을 밀어 주는 기능**을 붙인 저장소입니다.
소모품 교체 시기처럼 서버만 아는 사건을 앱이 계속 물어보지 않아도 받게 하는 것이 목적이었습니다.

> 두 저장소는 같은 코드베이스에서 갈라져 나왔습니다. **도메인 API의 설계 의도와 팀 내 역할 경계는
> [gildongE 저장소](https://github.com/jgjoe/gildonge-backend)에 정리돼 있고, 이 저장소는 알림 기능만 추가로 담습니다.**

---

## 주요 기능

| 영역 | 구현 내용 | 확인 위치 |
|---|---|---|
| 구독 | 사용자별 SSE 연결 수립, 연결 직후 확인 이벤트 발송 | `controller/AlertController` · `service/AlertServiceImpl` |
| 발송 | 사용자 단위로 알림 push | `AlertServiceImpl.pushAlert` |
| 저장 | 알림 이력을 MongoDB에 기록·조회 | `NotificationController` · `NotificationRepository` · `model/Notification` |
| 테스트 경로 | 특정 사용자에게 테스트 알림을 쏘는 엔드포인트 | `AlertController` `POST /alerts/test/{userId}` |
| **변경 감지** | MongoDB Change Stream을 구독해 소모품·주행패턴 문서가 바뀌면 즉시 알림 발생 | `ConsumableChangeStreamListener` · `DrivingPatternChangeStreamListener` |
| **정기 점검** | 매일 자정(Asia/Seoul)에 교체 시기가 다가온 소모품을 찾아 `CONSUMABLE_DUE_SOON` 알림 발송 | `ConsumableInspectionScheduler` |

## 설계 판단

### 폴링 대신 푸시를 골랐다

교체 시기 알림은 **언제 발생할지 서버만 압니다.** 앱이 주기적으로 물어보게 하면
서버는 대부분 "없다"고 답하는 요청을 계속 받고, 그렇다고 주기를 늘리면 알림이 늦습니다.

연결을 열어 두고 **서버가 사건이 생겼을 때 보내는 방식**을 골랐습니다.
알림은 서버 → 클라이언트 한 방향이면 충분해서 WebSocket 대신 **SSE(Server-Sent Events)**를 썼습니다.
HTTP 위에서 동작하고 브라우저의 `EventSource`로 바로 받을 수 있어 구현·운용 비용이 낮습니다.

```js
new EventSource("http://<서버주소>:8080/alerts/subscribe/USER123");
```

### 한 사용자가 여러 기기로 접속한다

emitter를 사용자당 하나로 두면 **폰과 태블릿 중 한쪽만 알림을 받습니다.**
사용자별로 emitter를 **목록으로 관리**해 열려 있는 연결 전부에 보내도록 했습니다.

### 끊어진 연결을 남겨두지 않는다

SSE 연결은 타임아웃·네트워크 오류·클라이언트 종료로 조용히 끊깁니다.
그대로 두면 죽은 emitter에 계속 쓰다가 예외가 나고 메모리도 샙니다.
**완료·타임아웃·오류 콜백에서 목록에서 제거**하고, 발송 중 실패한 emitter도 즉시 정리합니다.

### 타임아웃을 명시했다

무한 대기로 두면 끊긴 연결을 서버가 알아차릴 수단이 없습니다. 6시간 타임아웃을 걸어
주기적으로 재구독하게 했습니다.

### 알림이 생기는 경로를 둘로 나눴다

**연결만 열어 두면 보낼 것이 없습니다.** 알림이 언제 생기는지를 두 갈래로 만들었습니다.

- **즉시 반응** — MongoDB **Change Stream**을 구독해 소모품·주행패턴 문서가 바뀌는 순간 알림을 만듭니다. 앱이 폴링하지 않아도 서버가 먼저 압니다.
- **정기 점검** — 교체 시기처럼 "시간이 지나서" 생기는 사건은 변경 이벤트가 없습니다. 매일 자정에 도래 예정 소모품을 훑는 스케줄러를 따로 뒀습니다.

## 기술 스택

| 영역 | 기술 |
|---|---|
| 언어·프레임워크 | Java 17, Spring Boot 3 |
| 실시간 | Spring MVC `SseEmitter` (Server-Sent Events) |
| 데이터 | MongoDB, Spring Data MongoDB |

## 실행

환경변수로 접속 정보를 넘깁니다.

```text
MONGODB_URI=mongodb+srv://<user>:<password>@<cluster>/<db>
```

```bash
./gradlew bootRun
```

구독: `GET /alerts/subscribe/{userId}` · 테스트 발송: `POST /alerts/test/{userId}`

## 만든 사람

**Jigwan Joe** — Backend

- GitHub: [@jgjoe](https://github.com/jgjoe)
- Email: jigwan.joe@gmail.com
