# Slack ↔ Hermes 연동 구조 정리

> 작성일: 2026-09-13

## 목차

- [전체 흐름](#전체-흐름-블로그-세팅-기준-10단계)
- [Slack App과 Bot 개념 구분](#slack-app과-bot-개념-구분)
- [Socket Mode와 xapp- 토큰](#socket-mode와-xapp--토큰)
- [Bot Token Scopes — 권한](#bot-token-scopes--권한)
- [Event Subscriptions — 이벤트 구독](#event-subscriptions--이벤트-구독)
- [Install App → Bot Token 발급](#install-app--bot-token-발급)
- [Hermes 쪽 설정 (토큰 3종 + reply_in_thread)](#hermes-쪽-설정-토큰-3종--reply_in_thread)
- [Gateway 실행 → Bot 초대 → 테스트](#gateway-실행--bot-초대--테스트)
- [내가 연결을 실패했던 지점: Event 구독이 안 되어 있었음](#내가-연결을-실패했던-지점-event-구독이-안-되어-있었음)
- [내 환경 현재 상태 (실제 파일 기준)](#내-환경-현재-상태-실제-파일-기준)
- [참고 링크](#참고-링크)

## 전체 흐름

1. Slack App 만들기 (From scratch)
2. Information: App name, 설명, 아이콘, 배경색 설정
3. Bot Token Scopes: Bot이 할 수 있는 API 권한 설정
4. Socket Mode 켜고 xapp- 토큰(App-Level Token) 발급
5. Event Subscriptions: 어떤 이벤트일 때 Hermes에게 알릴지 구독 설정
6. Install App: 워크스페이스에 설치 → Bot Token(xoxb-) 발급
7. 앱 설치 확인 + 봇 멤버 ID 복사 (SLACK_ALLOWED_USERS)
8. Hermes 환경변수/설정 파일에 토큰 3종 입력
9. Gateway 실행 (Socket Mode 통로 유지)
10. 채널에서 `/invite @Hermes`로 봇 초대 → 멘션 테스트

> 출처 블로그: <https://velog.io/@yongukpark/Slack%EC%97%90%EC%84%9C-Hermes-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0-%EC%84%B8%ED%8C%85%ED%8E%B8>

## Slack App과 Bot 개념 구분

- **Slack App**: 외부 프로그램(Hermes)을 Slack 워크스페이스에 붙이기 위한 통로/컨테이너. 기능 자체와는 무관하고, 설정(scope/event/토큰)을 담는 그릇.
- **Bot**: 그 App 안에 존재하는, 실제로 메시지를 받고 보내는 주체. Hermes가 행동할 수 있는 환경이 App, 실제 Hermes가 Bot.

즉 **App = 설정 그릇**, **Bot = 행동 주체**로 나눠 이해하면 된다.

## Socket Mode와 xapp- 토큰

- 보통 Slack → 외부 서비스는 **웹훅 URL**로 요청을 보내는 구조인데, 로컬에서 Hermes를 돌리는 경우 공개 URL이 없어서 이 방식이 어렵다.
- **Socket Mode**를 켜면 Slack과 로컬 앱 사이에 WebSocket 통로가 계속 열려 있어서, 로컬 앱이 언제든 Slack의 요청을 받을 수 있다.
- 이 통로를 열기 위해 필요한 토큰이 **App-Level Token (xapp-)** 이다. Hermes 쪽에서는 `SLACK_APP_TOKEN`으로 사용한다.

## Bot Token Scopes — 권한

- Bot Token Scope는 **Bot이 Slack API로 무엇을 할 수 있는지** 정하는 OAuth 권한 목록이다.
- 용도별로 필요한 scope가 다르지만, Hermes를 혼자 쓰는 경우 큰 고민 없이 주는 방식도 가능하다. 다만 실제 Hermes가 제대로 동작하려면 아래 scope들이 필요하다.

### Hermes가 정상 동작하기 위해 필요한 대표 scope

- `app_mentions:read` — 멘션(app mention) 메시지를 읽을 수 있게 함
- `chat:write` — Bot이 채팅 메시지를 작성할 수 있게 함
- `commands` — 슬래시 커맨드(/hermes 등)를 받을 수 있게 함
- `im:read`, `im:write` — DM을 읽고 답할 수 있게 함
- `channels:read`, `groups:read`, `im:read`, `mpim:read` — 채널/그룹/DM 목록을 읽을 수 있게 함 (채널 디렉터리 구축에 필요)
- `channels:history`, `groups:history`, `im:history`, `mpim:history` — 해당 채널의 메시지 기록을 읽을 수 있게 함
- `files:read`, `files:write` — 파일 관련 처리
- `reactions:read` — 반응(이모지) 이벤트 수신
- `users:read` — 사용자 정보 읽기
- `assistant:write` — 어시스턴트 관련 쓰기

> 실제 필요한 scope 목록은 **Hermes가 생성하는 매니페스트**를 기준으로 확인하는 것이 정확하다. 스코프는 Hermes 버전마다 달라질 수 있으므로 외워서 등록하지 말고, `slack-manifest.json`의 `oauth_config.scopes.bot`를 본다.

## Event Subscriptions — 이벤트 구독

- Event Subscriptions는 **어떤 일이 일어났을 때 Slack이 Hermes에게 알려줄지**를 정하는 구독 목록이다.
- 스코프가 "봇이 할 수 있는 일"이라면, 이벤트는 "봇에게 알려줄 일"이다. 둘은 다르다.
  - 예: `chat:write` scope가 있어야 Bot이 메시지를 쓸 수 있지만, `message.channels` 이벤트가 구독되어 있어야 채널 메시지를 감지하고 반응을 시작할 수 있다.

### Hermes가 반응하기 위해 필요한 대표 이벤트

- `app_mention` — 누군가가 Bot을 멘션(@Hermes)했을 때. 가장 기본적이고 많이 쓰는 이벤트.
- `message.channels` — 채널 메시지
- `message.groups` — 그룹 메시지
- `message.im` — DM 메시지
- `message.mpim` — 멀티파티 DM 메시지
- `reaction_added`, `reaction_removed` — 이모지 반응 이벤트

블로그는 대부분 `app_mention`만으로도 편하게 쓴다고 적고 있다. 실제로 혼자 쓰는 용도면 `app_mention` 위주로 시작해도 된다. 다만 채널 메시지에도 반응하게 하려면 `message.channels` 등이 필요하다.

## Install App → Bot Token 발급

- App 설정을 마쳤으면 워크스페이스에 설치(Install App)한다.
- 설치 시 Bot Token(`xoxb-`로 시작)이 발급된다. 이 토큰으로 Bot이 해당 워크스페이스에 접근한다.
- Hermes 쪽에서는 `SLACK_BOT_TOKEN`으로 사용한다.
- 설치 완료 화면에 표시되는 권한 목록을 다시 확인할 수 있다.

## Hermes 쪽 설정 (토큰 3종 + reply_in_thread)

블로그 기준 준비해야 할 토큰 3종:

- `SLACK_APP_TOKEN` (xapp-): Socket Mode용 App-Level Token
- `SLACK_BOT_TOKEN` (xoxb-): 워크스페이스 접근용 Bot Token
- `SLACK_ALLOWED_USERS`: 봇을 사용할 수 있는 멤버 ID (워크스페이스라도 특정 유저만 접근 제어 가능)

내 환경(config.yaml 기준)에서는 추가로:

- `platforms.slack.reply_in_thread: true` — 답변을 스레드로 달게 하여 메인 메시지 탭이 복잡해지는 것을 막는다.
- 이 설정은 `hermes config set slack.reply_in_thread true`로 바꾸고, 게이트웨이 재시작이 필요하다.

## Gateway 실행 → Bot 초대 → 테스트

- **Gateway를 실행하지 않으면** Slack에서 아무리 메시지를 보내도 Hermes가 받지 못한다. Socket Mode 통로가 유지되어야 하기 때문이다.
- 봇은 채널별로 초대해야 해당 채널에서 사용할 수 있다. 채널에서 `/invite @Hermes` 또는 채널 → Add apps 로 초대.
- 첫 메시지는 `@멘션`이 있어야 Bot이 반응하는 것이 기본 동작이다.
- 홈 채널이 없으면 경고문이 뜰 수 있는데, `/sethome` 명령으로 지정한 채널을 기본 홈 채널로 설정하면 사라진다.

## 내가 연결을 실패했던 지점: Event 구독이 안 되어 있었음

- 연결이 안 됐던 원인은 **Event 연결이 안 되어 있었기 때문**이다.
- 즉 Bot Token Scope는 되어 있었거나, 토큰은 발급되어 있었지만, **Event Subscriptions 쪽에서 필요한 이벤트가 활성화/구독되어 있지 않아서** Slack이 Hermes에게 이벤트를 전달하지 못하는 상태였다.
- 이 경우 토큰이나 Socket Mode가 정상이어도 Bot은 메시지를 감지하지 못한다.
- 해결: Event Subscriptions에서 필요한 이벤트(`app_mention` 등)를 활성화했고, 그에 맞는 scope(`app_mentions:read` 등)가 Bot Token Scope에 들어 있는지 같이 확인했다.
- 정리: **토큰만 있다고 되는 게 아니고, "무엇을 구독할지"(Event)와 "구독한 이벤트를 읽을 권한"(Scope)이 둘 다 맞아야 한다.**

## 내 환경 현재 상태 (실제 파일 기준)

- Slack 앱 매니페스트: `~/.hermes/slack-manifest.json`
  - 봇 사용자 있음, Socket Mode ON
  - 주요 스코프: app_mentions:read, chat:write, commands, im:read, channels:read, groups:read, files:read/write, reactions:read, users:read 등
  - 이벤트 구독: app_mention, assistant_thread_context_changed, assistant_thread_started, message.channels, message.groups, message.im, message.mpim, reaction_added, reaction_removed
- 게이트웨이 상태: `gateway_state.json` → platforms.slack.state = "connected", 에러 없음
- 채널 디렉터리: `~/.hermes/channel_directory.json`
  - general (C077EGF25S9), hermes (C0C22RAUZB2)
- config.yaml: `platforms.slack.reply_in_thread = true`
- 현재 이 스레드 자체가 그 연결 통로로 들어온 메시지다.

## 참고 링크

- 블로그(원안): <https://velog.io/@yongukpark/Slack%EC%97%90%EC%84%9C-Hermes-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0-%EC%84%B8%ED%8C%85%ED%8E%B8>
- Hermes 공식 문서 (Slack): <https://hermes-agent.nousresearch.com/docs/user-guide/messaging/slack#step-2-configure-bot-token-scopes>
- Hermes GitHub: <https://github.com/nousresearch/hermes-agent>
- 내 매니페스트: `~/.hermes/slack-manifest.json`
- 내 게이트웨이 상태: `~/.hermes/gateway_state.json`
- 내 채널 디렉터리: `~/.hermes/channel_directory.json`

![](images/default.jpg)
#slack #hermes #봇연동
