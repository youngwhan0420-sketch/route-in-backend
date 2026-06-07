# ⚙ Route-In Backend

Route-In 서비스의 백엔드 서버입니다.

Spring Boot 기반 REST API와 WebSocket을 통해 게시글, 러닝코스, 루틴, 팔로우, 알림, 채팅, AI 추천, 인바디, 출석 기능을 제공합니다.

[🔗 Frontend 레포](https://github.com/Koreait-Triple-Stack/route_in_frontend.git)　|　[🔗 배포 주소](https://routein.store)

---

## 🛠 Tech Stack

| 구분 | 기술 |
|---|---|
| Framework | Spring Boot |
| ORM / SQL Mapper | MyBatis |
| Database | MySQL |
| 인증 | Spring Security · OAuth2 · JWT Bearer Token |
| 실시간 통신 | WebSocket · STOMP |
| Infra | Docker · Nginx · GCP · GitHub Actions |

---

## 📌 본인 담당 기능

- 출석 시스템 기능 설계 및 구현
- RESTful API 설계 및 구현
- Spring Boot Controller / Service / Repository / Mapper 계층 구조 구현
- MyBatis 기반 SQL 작성 및 DB 연동
- 인증 사용자 기준 API 처리
- 서버 데이터 기준 상태 동기화 구조 설계

---

# 1. Backend 전체 구조

Route-In 백엔드는 Controller에서 API 요청을 받고, Service에서 비즈니스 로직을 처리한 뒤 Repository와 Mapper를 통해 MySQL에 접근하는 구조입니다.

```text
Client
→ Spring Boot Controller
→ Service
→ Repository
→ MyBatis Mapper
→ MySQL
→ ApiRespDto 응답
```

---

## 1-1. 계층별 역할

| 계층 | 역할 |
|---|---|
| Controller | HTTP 요청 수신, 요청 데이터 추출, 인증 사용자 확인 |
| Service | 핵심 비즈니스 로직 처리 |
| Repository | Mapper 호출을 감싸는 데이터 접근 계층 |
| Mapper | MyBatis SQL 실행 |
| DB | 실제 데이터 저장 및 조회 |

---

## 1-2. 공통 응답 구조

프로젝트에서는 `ApiRespDto`를 사용해 API 응답 구조를 통일했습니다.

```java
public class ApiRespDto<T> {
    private String status;
    private String message;
    private T data;
}
```

응답 예시는 다음과 같습니다.

```json
{
  "status": "success",
  "message": "출석 월 조회",
  "data": [
    "2026-06-01",
    "2026-06-02"
  ]
}
```

프론트엔드는 `status` 값을 기준으로 성공 여부를 판단하고, `data` 값을 화면에 사용합니다.

---

# 2. 출석 시스템 기능 설계 및 구현

## 2-1. 관련 Backend 파일 구조

```text
src/main/java/com/triple_stack/route_in_backend/controller/AttendanceController.java
src/main/java/com/triple_stack/route_in_backend/service/AttendanceService.java
src/main/java/com/triple_stack/route_in_backend/repository/AttendanceRepository.java
src/main/java/com/triple_stack/route_in_backend/mapper/AttendanceMapper.java
src/main/resources/mappers/attendance_mapper.xml
```

---

## 2-2. 출석 시스템 전체 로직

```text
1. 사용자가 로그인 또는 새로고침 후 서비스에 접근
2. 프론트엔드에서 /user/account/principal API 요청
3. 백엔드 UserAccountController에서 로그인 사용자 정보 확인
4. AttendanceService.autoCheckToday() 실행
5. attendance_tb에 오늘 날짜 출석 데이터 저장 시도
6. 이미 출석한 경우 INSERT IGNORE로 중복 저장 방지
7. 오늘 출석 팝업을 이미 봤는지 popup_shown 값 조회
8. popup_shown = 0이면 principal.checked = true로 응답
9. 프론트엔드에서 principal.checked 값이 true이면 출석 팝업 표시
10. 사용자가 팝업을 닫으면 PATCH /attendance/popup/shown 요청
11. 서버에서 오늘 출석 row의 popup_shown 값을 1로 변경
12. 이후 같은 계정으로 다시 접속해도 팝업이 다시 뜨지 않음
```

---

## 2-3. 로그인 시 자동 출석 체크 로직

사용자 정보를 조회하는 `/user/account/principal` API에서 출석 체크를 함께 처리했습니다.

```java
@GetMapping("/principal")
public ResponseEntity<?> getPrincipal(@AuthenticationPrincipal PrincipalUser principalUser) {
    boolean checked = attendanceService.autoCheckToday(principalUser);
    if(principalUser != null) {
        principalUser.setChecked(checked);
    }
    return ResponseEntity.ok(new ApiRespDto<>("success", "회원 조회 완료", principalUser));
}
```

### 로직 설명

```text
/user/account/principal 요청
→ Spring Security가 JWT 검증
→ @AuthenticationPrincipal로 로그인 사용자 정보 주입
→ attendanceService.autoCheckToday(principalUser) 호출
→ 오늘 출석 데이터 저장
→ 오늘 출석 팝업 표시 여부 반환
→ principal.checked에 저장
→ 프론트엔드로 사용자 정보와 함께 응답
```

### 설계 이유

출석 체크를 별도의 버튼 클릭으로 처리하지 않고, 로그인 사용자 조회 흐름에 포함했습니다.

이렇게 하면 사용자가 메인 페이지에 접근하는 시점에 서버 기준으로 자동 출석 처리할 수 있습니다.

---

## 2-4. 하루 1회 출석 처리 로직

`AttendanceService.autoCheckToday()`에서 현재 로그인한 사용자의 `userId`를 기준으로 오늘 출석 데이터를 생성합니다.

```java
@Transactional
public boolean autoCheckToday(PrincipalUser principalUser){
    if (principalUser == null) return false;

    Integer userId = principalUser.getUserId();
    attendanceRepository.insertToday(userId);

    Integer shown = attendanceRepository.selectPopupShownToday(userId);
    return shown == 0;
}
```

### 내부 처리 흐름

```text
principalUser null 체크
→ 로그인 사용자가 없으면 false 반환
→ principalUser에서 userId 추출
→ attendanceRepository.insertToday(userId)
→ 오늘 출석 데이터 INSERT 시도
→ attendanceRepository.selectPopupShownToday(userId)
→ 오늘 팝업 표시 여부 조회
→ popup_shown이 0이면 true 반환
→ popup_shown이 1이면 false 반환
```

실제 SQL은 다음과 같습니다.

```sql
INSERT IGNORE INTO attendance_tb(user_id, attendance_date)
VALUES (#{userId}, CURDATE())
```

### 핵심 로직

`INSERT IGNORE`를 사용했기 때문에 같은 사용자가 같은 날짜에 다시 로그인해도 중복 출석 데이터가 생성되지 않습니다.

```text
첫 로그인
→ attendance_tb에 오늘 날짜 row 생성

같은 날 재접속
→ 같은 user_id + attendance_date 데이터가 이미 있음
→ INSERT IGNORE로 추가 저장 무시
```

즉, 출석 여부 판단 기준은 프론트엔드 상태가 아니라 DB에 저장된 `user_id + attendance_date` 조합입니다.

---

## 2-5. 출석 팝업 표시 여부 관리 로직

출석 팝업은 브라우저 localStorage가 아니라 DB의 `popup_shown` 값을 기준으로 관리합니다.

```sql
SELECT IFNULL(popup_shown, 0)
FROM attendance_tb
WHERE user_id = #{userId}
AND attendance_date = CURDATE()
```

사용자가 팝업을 닫으면 다음 API를 호출합니다.

```http
PATCH /attendance/popup/shown
```

백엔드에서는 오늘 출석 데이터의 `popup_shown` 값을 1로 변경합니다.

```java
@Transactional
public void markPopupShownToday(PrincipalUser principalUser) {
    if (principalUser == null) return;
    attendanceRepository.updatePopupShownToday(principalUser.getUserId());
}
```

실제 SQL은 다음과 같습니다.

```sql
UPDATE attendance_tb
SET popup_shown = 1
WHERE user_id = #{userId}
AND attendance_date = CURDATE()
AND IFNULL(popup_shown, 0) = 0
```

### 내부 처리 흐름

```text
사용자가 출석 팝업 닫기
→ PATCH /attendance/popup/shown 요청
→ 백엔드에서 현재 로그인 사용자 확인
→ 오늘 날짜 attendance row 조회
→ popup_shown 값 1로 변경
→ 이후 같은 날짜에는 팝업 미표시
```

### 설계 이유

localStorage는 브라우저 또는 기기별로 값이 따로 저장됩니다.

예를 들어 A기기에서 팝업을 닫아도 B기기에서는 다시 팝업이 뜰 수 있습니다.

이를 해결하기 위해 `popup_shown` 상태를 DB에 저장하고, 서버가 단일 기준으로 팝업 표시 여부를 판단하도록 구현했습니다.

---

## 2-6. 월별 출석 날짜 조회 로직

달력 화면에서는 월 단위로 출석 날짜 목록을 조회합니다.

```http
GET /attendance/month?ym=2026-06
```

Controller에서는 `ym` 파라미터와 인증 사용자를 받아 Service로 전달합니다.

```java
@GetMapping("/month")
public ResponseEntity<?> getMonth(@RequestParam String ym,
                                  @AuthenticationPrincipal PrincipalUser principalUser) {
    return ResponseEntity.ok(attendanceService.getMonthDates(ym, principalUser));
}
```

Service에서는 로그인 사용자가 없으면 예외를 발생시키고, `userId`와 `ym`을 DTO에 담아 조회합니다.

```java
public ApiRespDto<?> getMonthDates(String ym, PrincipalUser principalUser) {
    if(principalUser == null) throw new RuntimeException("로그인이 필요합니다.");

    AttendanceMonthReqDto attendanceMonthReqDto = new AttendanceMonthReqDto();
    attendanceMonthReqDto.setUserId(principalUser.getUserId());
    attendanceMonthReqDto.setYm(ym);

    List<String> dates = attendanceRepository.selectMonthDates(attendanceMonthReqDto);
    return new ApiRespDto<>("success", "출석 월 조회", dates);
}
```

실제 SQL은 해당 월의 1일부터 다음 달 1일 전까지의 날짜를 조회합니다.

```sql
SELECT DATE_FORMAT(attendance_date, '%Y-%m-%d')
FROM attendance_tb
WHERE user_id = #{userId}
AND attendance_date >= CONCAT(#{ym}, '-01')
AND attendance_date < DATE_ADD(CONCAT(#{ym}, '-01'), INTERVAL 1 MONTH)
ORDER BY attendance_date
```

### 내부 처리 흐름

```text
GET /attendance/month?ym=YYYY-MM 요청
→ 백엔드에서 로그인 사용자 확인
→ userId + ym으로 출석 날짜 목록 조회
→ ["2026-06-01", "2026-06-02"] 형태로 응답
→ 프론트엔드 달력에서 출석 날짜 강조 표시
```

---

# 3. RESTful API 설계 및 구현

## 3-1. API 설계 기준

Route-In 백엔드는 기능별로 Controller를 분리하고, 각 도메인에 맞는 URL Prefix를 사용했습니다.

```text
/user/account
/attendance
/board
/course
/routine
/follow
/comment
/chat
/notification
/inbody
/ai
/weather
```

### 설계 의도

```text
Controller 분리
→ 기능별 API 책임을 명확히 분리

DTO 사용
→ 요청 데이터 구조를 명확히 정의

Service 분리
→ 실제 비즈니스 로직을 Controller에서 분리

Repository / Mapper 분리
→ SQL 실행 책임 분리

ApiRespDto 사용
→ 프론트엔드에서 동일한 방식으로 응답 처리
```

---

## 3-2. REST API 공통 처리 로직

```text
1. React Page 또는 Component에서 사용자 이벤트 발생
2. API Service 함수 호출
3. Axios Instance에서 baseURL과 Authorization Header 적용
4. Spring Boot Controller에서 요청 수신
5. @AuthenticationPrincipal로 로그인 사용자 확인
6. @RequestBody, @RequestParam, @PathVariable로 요청 데이터 추출
7. Service에서 비즈니스 로직 처리
8. Repository를 통해 Mapper 호출
9. MyBatis XML에서 SQL 실행
10. DB 결과를 Service로 반환
11. ApiRespDto 형태로 프론트엔드에 응답
12. React Query에서 필요한 query invalidate 또는 재조회
13. 화면 최신 상태 반영
```

---

## 3-3. 인증 사용자 기반 API 처리 로직

백엔드에서는 `@AuthenticationPrincipal`을 사용해 현재 로그인 사용자를 가져옵니다.

```java
public ResponseEntity<?> addBoard(
    @RequestBody AddBoardReqDto addBoardReqDto,
    @AuthenticationPrincipal PrincipalUser principalUser
) {
    return ResponseEntity.ok(boardService.addBoard(addBoardReqDto, principalUser));
}
```

### 설계 이유

사용자 기준 API에서 클라이언트가 직접 `userId`를 보내면 조작 가능성이 있습니다.

그래서 게시글 작성, 출석 조회, 코스 저장처럼 현재 로그인 사용자가 중요한 기능은 JWT 인증 결과에서 사용자 정보를 가져오는 방식으로 처리했습니다.

```text
잘못된 방식
→ 클라이언트가 userId를 직접 보냄
→ 다른 userId로 요청 조작 가능

적용 방식
→ JWT에서 인증된 사용자 정보 사용
→ 서버가 userId를 결정
→ 데이터 조작 가능성 감소
```

---

## 3-4. 요청 데이터 분리 로직

REST API에서는 요청 목적에 따라 데이터를 받는 방식을 분리했습니다.

| 방식 | 사용 상황 | 예시 |
|---|---|---|
| `@PathVariable` | 특정 리소스 ID가 URL에 포함될 때 | `/board/{boardId}` |
| `@RequestParam` | 검색어, 월, 필터처럼 조회 조건일 때 | `/attendance/month?ym=2026-06` |
| `@RequestBody` | 등록, 수정처럼 복잡한 객체를 전달할 때 | `/board/add` |

---

# 4. 주요 REST API 엔드포인트

## 4-1. User / Account

| Method | Endpoint | 설명 |
|---|---|---|
| GET | `/user/account/principal` | 로그인 사용자 정보 조회 및 당일 출석 자동 체크 |
| GET | `/user/account/userId/{userId}` | userId 기준 사용자 조회 |
| GET | `/user/account/username/{username}` | username 기준 사용자 조회 |
| GET | `/user/account/duplicated/{username}` | username 중복 확인 |
| POST | `/user/account/change/address` | 주소 변경 |
| POST | `/user/account/change/username` | 사용자 이름 변경 |
| POST | `/user/account/change/profileImg` | 프로필 이미지 변경 |
| POST | `/user/account/change/bodyInfo` | 키 / 몸무게 변경 |
| POST | `/user/account/change/currentRun` | 현재 러닝 정보 변경 |
| POST | `/user/account/change/weeklyRun` | 주간 러닝 정보 변경 |
| POST | `/user/account/withdraw` | 회원 탈퇴 처리 |

## 4-2. Attendance

| Method | Endpoint | 설명 |
|---|---|---|
| GET | `/attendance/month?ym=YYYY-MM` | 특정 월 출석 날짜 목록 조회 |
| PATCH | `/attendance/popup/shown` | 오늘 출석 팝업 표시 완료 처리 |

## 4-3. Board

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/board/add` | 게시글 작성 |
| POST | `/board/update` | 게시글 수정 |
| POST | `/board/remove` | 게시글 삭제 |
| GET | `/board/list` | 게시글 목록 조회 |
| GET | `/board/list/infinite` | 게시글 무한스크롤 조회 |
| GET | `/board/{boardId}` | 게시글 상세 조회 |
| GET | `/board/user/{userId}` | 특정 사용자 게시글 조회 |
| GET | `/board/search?keyword=` | 게시글 검색 |
| POST | `/board/change/recommend` | 게시글 추천 / 추천 취소 |
| GET | `/board/recommend/{boardId}` | 게시글 추천 사용자 목록 조회 |
| POST | `/board/copy/payload` | 공유 루틴 / 코스 복사 |

## 4-4. Course

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/course/add` | 러닝 코스 저장 |
| GET | `/course/get/board/{boardId}` | 게시글에 연결된 코스 조회 |
| GET | `/course/get/user/{userId}` | 사용자 코스 목록 조회 |
| GET | `/course/get/favorite/{userId}` | 즐겨찾기 코스 조회 |
| POST | `/course/update` | 코스 수정 |
| POST | `/course/change/favorite` | 코스 즐겨찾기 변경 |
| GET | `/course/delete/{courseId}` | 코스 삭제 |

## 4-5. Routine

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/routine/add` | 루틴 등록 |
| POST | `/routine/update` | 루틴 수정 |
| POST | `/routine/get` | 루틴 조회 |
| POST | `/routine/remove` | 루틴 제거 |
| POST | `/routine/delete/{routineId}` | routineId 기준 루틴 삭제 |
| POST | `/routine/change/checked/{routineId}` | 루틴 완료 여부 변경 |

## 4-6. Follow

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/follow/add` | 팔로우 추가 |
| POST | `/follow/delete` | 팔로우 삭제 |
| POST | `/follow/change` | 팔로우 / 언팔로우 토글 |
| GET | `/follow/status` | 두 사용자 간 팔로우 여부 조회 |
| GET | `/follow/follower/{userId}` | 팔로워 목록 조회 |
| GET | `/follow/following/{userId}` | 팔로잉 목록 조회 |

## 4-7. Comment

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/comment/add` | 댓글 / 대댓글 작성 |
| GET | `/comment/{boardId}` | 게시글 댓글 목록 조회 |
| GET | `/comment/delete/{commentId}` | 댓글 삭제 |

## 4-8. Chat

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/chat/room/add` | 채팅방 생성 |
| GET | `/chat/room/list/{userId}` | 사용자 채팅방 목록 조회 |
| GET | `/chat/room/{roomId}` | 채팅방 상세 조회 |
| POST | `/chat/room/quit` | 채팅방 나가기 |
| POST | `/chat/room/change/title` | 채팅방 제목 변경 |
| POST | `/chat/room/participant/add` | 채팅방 참가자 추가 |
| POST | `/chat/message/add` | 메시지 저장 |
| POST | `/chat/message/change` | 메시지 수정 |
| POST | `/chat/message/delete/{messageId}` | 메시지 삭제 |
| GET | `/chat/message/list` | 메시지 무한스크롤 조회 |
| GET | `/chat/message/unread/{userId}` | 읽지 않은 채팅 수 조회 |
| POST | `/chat/mute/notification` | 채팅 알림 음소거 |
| POST | `/chat/change/favorite` | 채팅방 즐겨찾기 변경 |
| POST | `/chat/read/room` | 채팅방 읽음 처리 |

## 4-9. Notification

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/notification/add` | 알림 생성 |
| GET | `/notification/get/list/{userId}` | 사용자 알림 목록 조회 |
| POST | `/notification/delete/{notificationId}` | 알림 단건 삭제 |
| POST | `/notification/delete/all/{userId}` | 사용자 알림 전체 삭제 |
| GET | `/notification/count/{userId}` | 읽지 않은 알림 개수 조회 |

## 4-10. InBody

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/inbody/add` | 인바디 기록 등록 |
| POST | `/inbody/delete` | 인바디 기록 삭제 |
| GET | `/inbody/list/{userId}` | 사용자 인바디 기록 조회 |

## 4-11. AI Recommend

| Method | Endpoint | 설명 |
|---|---|---|
| GET | `/ai/chatList/{userId}` | 사용자 AI 질문 / 답변 목록 조회 |
| GET | `/ai/recommend/{userId}` | 오늘의 AI 추천 조회 |
| POST | `/ai/question` | AI 질문 전송 및 응답 생성 |
| GET | `/ai/recommend/course` | 추천 코스 목록 조회 |

## 4-12. Weather

| Method | Endpoint | 설명 |
|---|---|---|
| GET | `/weather?lat=&lon=` | 위도 / 경도 기준 날씨 조회 |

---

# 5. 대표 기능별 Backend 로직

## 5-1. 게시글 작성 API 로직

```text
POST /board/add
→ Controller에서 AddBoardReqDto와 PrincipalUser 수신
→ Service에서 게시글 타입 확인
→ board_tb에 게시글 기본 정보 저장
→ 타입이 ROUTINE이면 routine_tb 저장
→ 타입이 COURSE이면 course_tb와 course_point_tb 저장
→ ApiRespDto로 처리 결과 반환
→ 프론트엔드에서 게시글 목록 또는 상세 데이터 재조회
```

---

## 5-2. 추천 API 로직

```text
POST /board/change/recommend
→ userId와 boardId 확인
→ 현재 사용자가 해당 게시글을 추천했는지 조회
→ 추천하지 않았다면 recommend_tb 추가
→ 이미 추천했다면 recommend_tb 삭제
→ 게시글 상세 query 재조회
→ 추천 수와 버튼 상태 갱신
```

---

## 5-3. 팔로우 API 로직

```text
POST /follow/change
→ 로그인 사용자 확인
→ 대상 사용자 확인
→ follow_tb에 팔로우 관계가 있는지 조회
→ 없으면 팔로우 추가
→ 있으면 언팔로우 처리
→ 팔로워 수, 팔로잉 수, 버튼 상태 재조회
```

---

## 5-4. 러닝코스 저장 API 로직

```text
POST /course/add
→ 코스 기본 정보 수신
→ course_tb에 코스 정보 저장
→ 생성된 course_id 확보
→ 좌표 목록을 순서대로 course_point_tb에 저장
→ 저장 완료 응답
```

---

## 5-5. 출석 API 로직

```text
GET /user/account/principal
→ 로그인 사용자 정보 조회
→ autoCheckToday() 실행
→ 오늘 출석 row 생성
→ popup_shown 조회
→ principal.checked 값으로 팝업 여부 전달

GET /attendance/month?ym=YYYY-MM
→ userId와 ym 기준으로 해당 월 출석 날짜 조회
→ 날짜 배열 반환
→ Calendar에서 출석 날짜 강조

PATCH /attendance/popup/shown
→ 오늘 attendance row의 popup_shown = 1
→ 같은 날짜에 팝업 재노출 방지
```

---

# 6. 기술적 의사결정

## 6-1. 출석 중복 처리 — DB 기준 처리

| 방식 | 검토 내용 |
|---|---|
| 프론트 상태 기준 | localStorage 삭제, 브라우저 변경, 멀티 디바이스 환경에서 출석 정보 불일치 가능 |
| DB 기준 처리 ✅ | `attendance_tb`에 사용자별 날짜를 저장해 서버가 출석 여부 판단 |

출석은 사용자 기록 데이터이므로 브라우저 상태가 아니라 DB를 기준으로 처리했습니다.

---

## 6-2. 하루 1회 출석 — `INSERT IGNORE` 사용

```sql
INSERT IGNORE INTO attendance_tb(user_id, attendance_date)
VALUES (#{userId}, CURDATE())
```

동일한 사용자가 같은 날짜에 여러 번 로그인해도 중복 출석이 생성되지 않도록 설계했습니다.

---

## 6-3. 팝업 표시 상태 — `popup_shown` 서버 저장

| 방식 | 검토 내용 |
|---|---|
| localStorage | 기기별 저장이라 멀티 디바이스에서 상태 불일치 발생 |
| DB `popup_shown` ✅ | 사용자 + 날짜 기준으로 서버가 팝업 표시 여부를 단일 판단 |

출석 팝업은 사용자 경험과 관련된 상태이지만, 멀티 디바이스 환경에서는 서버 기준 관리가 더 안정적이라고 판단했습니다.

---

# 7. 트러블슈팅

## 7-1. 출석 팝업 멀티 디바이스 동기화 문제

### 문제

A기기에서 출석 팝업을 닫아도 B기기에서 다시 팝업이 표시될 수 있었습니다.

### 원인

팝업 표시 여부를 localStorage로 관리하면 브라우저 또는 기기 단위로 값이 저장됩니다.

### 해결

`attendance_tb`에 `popup_shown` 값을 저장하고, 서버에서 오늘 팝업 표시 여부를 판단하도록 변경했습니다.

### 결과

여러 기기에서 접속해도 사용자 계정과 날짜 기준으로 팝업 표시 상태가 일관되게 유지되었습니다.

---

## 7-2. 출석 중복 저장 문제

### 문제

사용자가 하루에 여러 번 로그인할 경우 같은 날짜 출석 데이터가 중복 저장될 수 있었습니다.

### 원인

출석 체크가 로그인 사용자 조회 시 자동 실행되기 때문에 같은 날 여러 번 호출될 수 있습니다.

### 해결

`attendance_tb`에 사용자와 날짜 기준 중복 방지 구조를 두고, SQL에서는 `INSERT IGNORE`를 사용했습니다.

### 결과

같은 사용자가 같은 날짜에 여러 번 로그인해도 출석 데이터는 한 번만 저장됩니다.

---

# 8. Database

## 8-1. 출석 테이블

| 컬럼 | 설명 |
|---|---|
| user_id | 출석한 사용자 ID |
| attendance_date | 출석 날짜 |
| popup_shown | 해당 날짜 출석 팝업 표시 여부 |

```text
user_id + attendance_date
→ 하루 1회 출석 판단 기준

popup_shown
→ 오늘 출석 팝업을 이미 보여줬는지 판단하는 기준
```

---

## 8-2. 주요 테이블

| 테이블 | 설명 |
|---|---|
| user_tb | 사용자 정보, OAuth2 사용자 정보 |
| board_tb | 게시글 정보 |
| comment_tb | 댓글 / 대댓글 |
| recommend_tb | 게시글 추천 |
| course_tb | 러닝 코스 기본 정보 |
| course_point_tb | 러닝 코스 좌표 정보 |
| routine_tb | 운동 루틴 정보 |
| follow_tb | 팔로우 관계 |
| room_tb | 채팅방 정보 |
| room_participant_tb | 채팅방 참가자 정보 |
| room_read_tb | 사용자별 채팅방 읽음 상태 |
| message_tb | 채팅 메시지 |
| notification_tb | 알림 정보 |
| ai_question_tb | AI 질문 이력 |
| ai_recommend_tb | AI 추천 응답 이력 |
| in_body_tb | 인바디 기록 |
| attendance_tb | 출석 기록 |

---

# 9. Backend 핵심 포인트

Route-In 백엔드에서 중요한 부분은 사용자의 행동이 발생했을 때 서버 데이터와 프론트엔드 화면 상태를 일치시키는 것이었습니다.

특히 출석 시스템은 사용자의 접속 상태와 날짜 기준 데이터가 함께 사용되기 때문에 프론트엔드 localStorage만으로 관리하면 데이터 정합성을 보장하기 어렵습니다.

그래서 출석 기록과 팝업 표시 여부를 모두 DB 기준으로 관리했고, REST API는 도메인별 Controller로 분리하고, 요청 데이터는 DTO로 전달하며, 응답은 `ApiRespDto` 구조로 통일했습니다.
