# Melissa

> AI와의 대화를 통해 하루를 돌아보고, 대화 내용을 일기로 정리해 주는 Android 마음 정리 다이어리

Melissa는 일기를 꾸준히 쓰기 어렵다는 문제를 해결하기 위해 만든 Android 앱입니다.
사용자가 직접 긴 글을 작성하는 대신 AI와 대화하며 하루와 감정을 정리하면,
대화 내용을 바탕으로 제목, 주요 활동, 만족도, 다음 날 계획이 포함된 일기를 생성합니다.
생성된 일기는 날짜별로 저장되며 달력에서 다시 확인할 수 있습니다.

## 개발 정보

| 항목 | 내용 |
| --- | --- |
| 개발 기간 | 2024.10.30 - 2024.12.04 (약 5주) |
| 프로젝트 형태 | 모바일 프로그래밍 수업 팀 프로젝트 |
| 주요 언어 | Java |
| 플랫폼 | Android (minSdk 26, targetSdk 32) |
| 담당 영역 | 앱 구조 설계, 핵심 기능 대부분 구현, API 연동, 기능 통합 및 프로젝트 조율 |

## 핵심 기능

### 대화형 하루 기록

- AI가 먼저 질문을 건네고 사용자의 답변을 바탕으로 후속 대화를 진행합니다.
- 사용자와 AI의 메시지를 역할별 UI로 구분하여 표시합니다.
- 같은 날에는 기존 대화 Thread를 이어서 사용할 수 있습니다.

### AI 일기 생성

- 사용자의 대화 내용을 GPT에 전달하여 하루를 구조화된 JSON으로 요약합니다.
- 제목, 주요 활동, 감정, 만족도, 다음 날 계획을 HTML 형식으로 생성합니다.
- 요약 결과와 전체 대화를 SQLite에 날짜별로 저장합니다.

### 달력 기반 기록 조회

- `ViewPager2`를 이용해 월 단위 달력을 양방향으로 탐색할 수 있습니다.
- 기록이 있는 날짜에는 AI가 생성한 일기 제목을 표시합니다.
- 날짜를 선택하면 요약 일기와 당시의 전체 대화를 다시 볼 수 있습니다.

### 웨이크워드

- Picovoice Porcupine의 한국어 모델을 적용했습니다.
- 메인 화면에서 "멜리사"를 감지하면 하단 버튼을 누른 것처럼 채팅 화면을 실행합니다.

## 기술 스택

| 구분 | 기술 |
| --- | --- |
| Language | Java 8 |
| Android | Android SDK, AppCompat, Material Components |
| UI | RecyclerView, ViewPager2, Fragment, GridLayout |
| Network | Retrofit2, OkHttp3, Gson |
| AI | OpenAI Assistants API v2, Chat Completions API |
| Voice | Picovoice Porcupine |
| Storage | SQLite, SharedPreferences |
| Build | Gradle 7.4, Android Gradle Plugin 7.3.0 |

## 동작 흐름

```text
웨이크워드 또는 버튼
        |
        v
채팅 화면 진입
        |
        v
Thread 조회 또는 생성
        |
        v
사용자 Message 전송
        |
        v
Run 생성 -> 상태 Polling -> Assistant 응답 조회
        |
        v
대화 내용 요약 요청
        |
        v
요약 JSON + 전체 대화를 SQLite에 저장
        |
        v
달력에서 날짜별 일기 조회
```

## 가장 집중해서 구현한 부분

### Assistants API 전용 클라이언트와 대화 오케스트레이션

이 프로젝트에서 가장 많은 시간과 노력을 들인 부분은 당시 베타였던
OpenAI Assistants API v2를 Android 앱에 통합하는 작업이었습니다.

일반적인 단일 요청·응답 방식과 달리 Assistants API는 다음 과정을 각각 관리해야 했습니다.

1. 대화 상태를 보관할 Thread 생성
2. Thread에 사용자 Message 추가
3. Assistant가 응답을 생성하도록 Run 실행
4. Run 상태를 주기적으로 조회
5. 실행 완료 후 Thread의 Message 목록 조회
6. API 응답을 앱의 `ChatMessage` 모델로 변환

이를 위해 Retrofit 엔드포인트만 선언하는 데 그치지 않고 다음 계층을 직접 구성했습니다.

- `GptApiService`: Thread, Message, Run 관련 HTTP 엔드포인트 정의
- `RetrofitClient`: 인증 헤더, 베타 버전 헤더, 타임아웃과 로깅 설정
- `ChatApiManager`: 저수준 API 호출을 앱에서 사용하기 쉬운 비동기 메서드로 추상화
- `ThreadManager`: Thread ID의 생성, 로컬 저장, 초기화 시점 관리
- `ChatActivity`: Message 전송부터 Run Polling과 UI 갱신까지 전체 흐름 조율

별도 배포 가능한 범용 SDK 형태는 아니지만, 여러 저수준 API를 조합해 Android 앱에 맞는
**경량 Assistants API 래퍼와 오케스트레이션 계층**을 직접 구현한 경험입니다.

## 개발 과정에서 어려웠던 점

### 비동기 상태 관리

메시지를 보낸 직후 답변이 반환되지 않기 때문에 `threadId`와 `runId`를 유지하고,
Run이 완료될 때까지 상태를 반복해서 확인해야 했습니다. 네트워크 요청의 순서를 보장하면서
UI가 먼저 갱신되거나 응답이 누락되지 않도록 콜백 흐름을 설계하는 것이 가장 어려웠습니다.

### 베타 API 직접 연동

개발 당시 Assistants API v2는 비교적 새롭게 공개된 베타 API였습니다.
Java Android 환경에서 바로 적용할 수 있는 예제가 충분하지 않아 공식 API 구조와 JSON 응답을
분석하고 Retrofit 인터페이스와 파싱 로직을 직접 작성했습니다.

### 대화 생명주기 정의

사용자가 앱을 다시 실행해도 같은 날의 대화를 이어가되, 새로운 하루에는 새 대화를 시작해야
했습니다. 이를 위해 `SharedPreferences`에 Thread ID와 생성 시간을 저장하고 매일 오전 4시
30분을 기준으로 Thread를 갱신하도록 구현했습니다.

### AI 결과를 앱 데이터로 연결

자유 형식인 AI 응답을 그대로 저장하면 달력과 상세 화면에서 일관되게 표시하기 어렵습니다.
따라서 출력 형식을 `title`과 `summary`를 가진 JSON으로 제한하고, 요약 본문은 HTML로 생성해
SQLite 저장과 Android 화면 렌더링까지 연결했습니다.

### 음성 기능의 Android 생명주기 처리

웨이크워드 기능에는 마이크 권한뿐 아니라 Activity 상태에 따른 리소스 관리가 필요했습니다.
`onStart`, `onStop`, `onDestroy`에서 Porcupine의 시작, 중지, 해제를 각각 처리했습니다.

## 얻은 경험과 인사이트

- AI 기능도 결국 명확한 상태와 데이터 흐름을 가진 소프트웨어 시스템으로 설계해야 합니다.
- 비동기 API에서는 성공 경로뿐 아니라 실패, 타임아웃, 취소와 재시도 정책이 중요합니다.
- 모델의 성능만큼 출력 형식과 파싱 가능한 계약을 설계하는 것이 제품 안정성에 영향을 줍니다.
- API 응답, 로컬 데이터베이스, 화면 상태를 하나의 사용자 흐름으로 연결하는 통합 경험을 얻었습니다.
- 프로토타입을 완성한 뒤에는 보안, 테스트, 오류 복구가 운영 가능한 제품과의 가장 큰 차이라는 점을 배웠습니다.
- 새로운 API를 사용할 때는 SDK에 의존하기보다 HTTP 규격과 상태 모델을 이해해야 문제를 해결할 수 있습니다.

## 프로젝트 구조

```text
app/src/main/java/com/example/melissa/
├── activities/   # 달력, 채팅, 일기 및 전체 대화 화면
├── adapters/     # RecyclerView와 ViewPager2 어댑터
├── database/     # SQLite 생성, 저장 및 조회
├── fragments/    # 월별 달력 화면
├── models/       # 채팅 메시지 모델과 JSON 변환
├── network/      # OpenAI API 클라이언트
└── utils/        # Thread 및 FloatingActionButton 관리
```

## 실행 방법

### 요구 환경

- Android Studio
- JDK 17
- Android SDK 32

### 로컬 설정

프로젝트 루트에 `local.properties`를 만들고 다음 값을 설정합니다.

```properties
sdk.dir=/path/to/Android/sdk
OPENAI_API_KEY=your_openai_api_key
PORCUPINE_ACCESS_KEY=your_picovoice_access_key
ASSISTANT_ID=your_openai_assistant_id
```

빌드와 단위 테스트는 다음 명령으로 실행할 수 있습니다.

```bash
./gradlew testDebugUnitTest
./gradlew assembleDebug
```

## 현재 한계와 개선 방향

이 저장소는 2024년에 만든 학습용 프로토타입으로, 현재 기준의 운영 환경을 목표로 하지는 않습니다.

- API 키가 `BuildConfig`를 통해 APK에 포함되므로 실제 서비스에서는 백엔드 프록시가 필요합니다.
- Run의 실패, 취소, 만료 상태와 Polling 타임아웃 처리를 추가해야 합니다.
- API 실패 시 버튼과 화면 상태를 복구하는 공통 오류 처리 구조가 필요합니다.
- 현재 테스트는 기본 예제 수준이므로 API, 데이터베이스, 주요 사용자 흐름에 대한 테스트가 필요합니다.
- SQLite 접근 계층을 통합하고 Room 등 구조화된 저장소로 전환할 수 있습니다.
- OpenAI Assistants API는 현재 deprecated 상태이며 2026년 8월 26일 종료 예정입니다.
  계속 운영하려면 Responses API와 Conversations API 기반으로 마이그레이션해야 합니다.

## 참고 자료

- [OpenAI Assistants API](https://platform.openai.com/docs/assistants/deep-dive)
- [Assistants API Migration Guide](https://platform.openai.com/docs/assistants/migration)
- [Picovoice Porcupine](https://picovoice.ai/platform/porcupine/)
