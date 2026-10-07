# Codex `config.toml` 주요 설정 가이드

확인일: **2026-10-07** · OpenAI 공식 문서 기준

로컬 Codex 사용자가 자주 조정하는 설정을 골라 정리했다. 전체 옵션 목록은 아니며, CLI·IDE·데스크톱 앱 및 설치 버전에 따라 지원 범위가 다를 수 있다. 특히 `[tui]`는 CLI 터미널 화면용이다. 아래의 **예시·시작값은 제품 기본값과 다르다.** 명시하지 않은 기본값은 모델이나 클라이언트 설정에 맡긴다.

## 먼저 보기: 가장 자주 바꾸는 핵심 키

아래는 사용 빈도 통계가 아니라, 일반적인 로컬 개발에서 먼저 살펴볼 설정을 추린 요약이다. 예시 값은 필요에 맞게 선택한다.

| 바꾸고 싶은 동작 | 핵심 키 | 선택 가능한 값 | 시작 예시 |
| --- | --- | --- | --- |
| 기본 모델 | `model` | 계정에서 사용 가능한 모델 ID | `"gpt-6.1-sol"` — 접근 가능할 때 |
| 추론 강도 | `model_reasoning_effort` | 모델별 지원 값: `low`, `medium`, `high`, `xhigh`, `max`, `ultra` 등 | `"medium"` |
| 답변 길이·상세도 | `model_verbosity` | `"low"`, `"medium"`, `"high"` — 지원 모델에 한함 | `"medium"` |
| 승인 요청 방식 | `approval_policy` | `"on-request"`, `"never"`, 세부 `granular` 객체 | `"on-request"` |
| 명령의 파일 접근 범위 | `sandbox_mode` | `"read-only"`, `"workspace-write"`, `"danger-full-access"` | `"workspace-write"` |
| 웹 검색 방식 | `web_search` | `"disabled"`, `"cached"`, `"indexed"`, `"live"` | `"cached"` |
| 명령 실행 중 인터넷 사용 | `sandbox_workspace_write.network_access` | `true`, `false` | `false` |
| 작업 완료·승인 알림 | `tui.notifications` | `true`, `false`, 이벤트 이름 배열 | `true` |

처음에는 **모델·추론 강도·승인 정책·샌드박스·웹 검색**부터 정하면 된다. 세부 설명은 아래 항목별 표를, 완성된 TOML 예제는 **9절**을 참고한다.

근거: [공식 문서의 자주 쓰는 설정](https://learn.chatgpt.com/docs/config-file/config-basic#common-configuration-options), [설정 레퍼런스](https://learn.chatgpt.com/docs/config-file/config-reference).

## 1. 설정파일 위치와 우선순위

### 어디에 저장할까?

| 범위 | 위치 / 사용법 |
| --- | --- |
| 사용자 공통 | `~/.codex/config.toml` |
| 특정 프로젝트 | `<프로젝트>/.codex/config.toml` — 신뢰한 프로젝트에서만 로드 |
| 이름 있는 프로필 | `~/.codex/<이름>.config.toml` → `codex --profile <이름>` |
| 한 번만 변경 | `codex -c '키=값'` |

`~`는 사용자 홈 디렉터리다. 이 작업 환경에서 사용자 공통 설정의 기본 경로는 `/Users/moonnjangho/.codex/config.toml`, 현재 프로젝트 설정 경로는 `/Users/moonnjangho/Documents/ChatGPT/harness-engineering/.codex/config.toml`이다. 이는 위치 안내이며 파일이 실제로 존재한다는 뜻은 아니다. `CODEX_HOME`을 지정했다면 사용자 설정은 `$CODEX_HOME/config.toml`에서 읽는다.

모든 프로젝트에 같은 취향을 적용하려면 **사용자 공통 설정**, 저장소에만 적용하려면 **프로젝트 설정**, 작업 종류별로 전환하려면 **프로필**, 이번 실행만 바꾸려면 **CLI 옵션**을 사용한다. IDE 확장에서는 톱니바퀴 → **Codex Settings → Open config.toml**로 사용자 설정을 열 수 있다.

### 같은 키가 여러 곳에 있으면 무엇이 이길까?

아래 표의 **숫자가 작을수록 우선순위가 높다.** 파일 하나를 통째로 선택하는 것이 아니라 같은 키의 값을 높은 우선순위 설정으로 덮어쓴다.

| 우선순위 | 설정 출처 | 경로 / 설명 |
| --- | --- | --- |
| 1 — 가장 높음 | CLI 플래그·`--config` | `--model`, `-c 'web_search="live"'` 등 |
| 2 | 프로젝트 설정 | 프로젝트 루트부터 현재 작업 디렉터리까지의 `.codex/config.toml`. 현재 디렉터리에 가까울수록 우선, 신뢰한 프로젝트만 적용 |
| 3 | 선택한 프로필 | `--profile research`로 선택한 `~/.codex/research.config.toml` |
| 4 | 사용자 공통 설정 | `~/.codex/config.toml` |
| 5 | 클라우드 관리 기본 설정 | 로그인한 워크스페이스에서 제공하는 경우 |
| 6 | 시스템 설정 | Unix에서는 `/etc/codex/config.toml`, 파일이 있는 경우 |
| 7 — 가장 낮음 | 내장 기본값 | 위 출처에서 지정하지 않은 값 |

**충돌 예시:** 사용자 설정에 `web_search = "cached"`, 선택한 프로필에 `"live"`, 신뢰한 프로젝트 설정에 `"disabled"`가 있으면 최종 값은 `"disabled"`다. 같은 실행에서 `-c 'web_search="live"'`를 지정하면 CLI 값인 `"live"`가 우선한다.

조직의 `requirements.toml` 등 **강제 정책은 이 표와 별도의 제약**이다. 예를 들어 조직이 `live` 검색을 허용하지 않으면 CLI 우선순위가 높아도 이를 우회하지 못한다. 프로젝트 설정에서 허용되지 않는 공급자·알림 관련 키도 있으므로 **5절**의 제한을 함께 확인한다.

### TOML 작성 시 주의할 점

TOML 문자열은 따옴표로 감싸고, 불리언은 `true`·`false`, 목록은 `["a", "b"]`로 작성한다. 최상위 키는 `[섹션]`보다 앞에 놓는다. 예를 들어 `[history]` 아래에 `model`을 쓰면 최상위 모델 설정이 아니다.

근거: [설정 기본](https://learn.chatgpt.com/docs/config-file/config-basic), [고급 설정](https://learn.chatgpt.com/docs/config-file/config-advanced).

## 2. 모델·추론·응답 스타일

| 키 | 값 / 옵션 | 의미 및 선택 기준 |
| --- | --- | --- |
| `model` | 사용 가능한 모델 ID 문자열 | 기본 모델. 공식 예시는 `"gpt-6.1-sol"`이지만 계정·워크스페이스 접근 권한 확인 필요 |
| `review_model` | 모델 ID 문자열 | `/review`용 모델. 생략하면 현재 세션 모델 |
| `model_reasoning_effort` | 모델이 제공하는 문자열. 예: `"low"`, `"medium"`, `"high"`, `"xhigh"`, `"max"`, `"ultra"` | 추론 강도. 모든 모델이 모든 값을 지원하는 것은 아님 |
| `model_reasoning_summary` | `"auto"`, `"concise"`, `"detailed"`, `"none"` | 추론 요약 표시 수준. `none`은 요약 비활성화 |
| `model_verbosity` | `"low"`, `"medium"`, `"high"` | 응답 상세도. 지원되는 Responses API 모델용이며 미지정 시 모델·프리셋 기본값 |
| `personality` | `"none"`, `"friendly"`, `"pragmatic"` | 지원 모델의 기본 말투 |
| `model_provider` | 공급자 ID 문자열, 기본 `"openai"` | 다른 공급자·게이트웨이를 사용할 때 변경 |
| `oss_provider` | `"ollama"`, `"lmstudio"` | CLI `--oss` 실행 시 사용할 로컬 공급자 |

`model_context_window`와 `model_auto_compact_token_limit`은 토큰 수를 받는 고급 설정이다. 후자는 자동 대화 압축 시점이며, 생략하면 모델 기본값을 쓴다. 큰 값을 입력한다고 모델의 실제 컨텍스트 용량이 늘어나는 것은 아니므로 처음에는 생략하는 편이 좋다.

근거: [설정 레퍼런스](https://learn.chatgpt.com/docs/config-file/config-reference), [모델·응답 설정](https://learn.chatgpt.com/docs/config-file/config-advanced#model-reasoning-verbosity-and-limits).

## 3. 승인과 샌드박스 — 먼저 이해할 항목

승인 정책은 **언제 승인을 요청할지**, 샌드박스는 **명령이 접근할 수 있는 범위**를 정한다.

| 키 | 옵션 | 의미 |
| --- | --- | --- |
| `approval_policy` | `"on-request"` | 필요하다고 판단한 작업에 승인 요청 가능 |
| ↳ | `"never"` | 승인 요청하지 않음. 샌드박스 제한을 자동 해제하는 설정은 아님 |
| ↳ | `{ granular = { ... } }` | 승인 요청 종류별 허용·자동 거절 설정 |
| `approvals_reviewer` | `"user"`(기본), `"auto_review"` | 승인 검토 주체. 자동 검토도 샌드박스 자체를 바꾸지는 않음 |
| `sandbox_mode` | `"read-only"` | 파일 쓰기를 제한하는 모드 |
| ↳ | `"workspace-write"` | 작업공간 등 허용된 루트에 쓰기 가능 |
| ↳ | `"danger-full-access"` | Codex 샌드박스 제한을 해제하는 모드. 실행 환경의 다른 제약은 별개 |

일반적인 로컬 개발의 **시작 예시**는 `on-request` + `workspace-write`다. 이는 이 가이드의 제안이며 제품 기본값 선언이 아니다.

`[sandbox_workspace_write]`에서 조정할 수 있는 주요 값:

| 키 | 값 | 의미 |
| --- | --- | --- |
| `network_access` | `true` / `false` | 샌드박스 명령의 외부 네트워크 허용 여부 |
| `writable_roots` | 경로 문자열 배열 | 추가 쓰기 허용 디렉터리 |
| `exclude_tmpdir_env_var` | `true` / `false` | `true`이면 `$TMPDIR`를 기본 쓰기 허용 대상에서 제외 |
| `exclude_slash_tmp` | `true` / `false` | `true`이면 `/tmp`를 기본 쓰기 허용 대상에서 제외 |

웹 검색·MCP의 네트워크와 명령 실행의 네트워크는 별개다. `network_access = false`만으로 모든 외부 도구가 차단된다고 해석하면 안 된다. 쓰기 가능한 작업공간 안에서도 `.git`, `.codex` 같은 보호 경로에는 추가 제한이 적용될 수 있다.

세부 승인 정책이 필요한 경우의 별도 예시:

```toml
approval_policy = { granular = { sandbox_approval = true, rules = true, mcp_elicitations = true, request_permissions = true, skill_approval = true } }
```

각 값의 `true`는 해당 **승인 요청을 표시할 수 있음**, `false`는 해당 요청을 **자동 거절**한다는 뜻이다. 자동 승인 스위치가 아니다. 각각 샌드박스 권한 확대, 명령 규칙, MCP 추가 입력, 권한 요청 도구, 스킬 승인을 제어한다.

근거: [승인·샌드박스 안내](https://learn.chatgpt.com/docs/agent-approvals-security), [설정 레퍼런스](https://learn.chatgpt.com/docs/config-file/config-reference).

## 4. 웹 검색

| `web_search` 값 | 동작 |
| --- | --- |
| `"disabled"` | 웹 검색 도구 비활성화 |
| `"cached"` | OpenAI가 유지하는 인덱스 사용, 외부 웹 직접 접근 없음. 일반 기본값 |
| `"indexed"` | 검색 인덱스가 허용하는 범위에서 외부 접근 |
| `"live"` | 실시간 웹 조회 |

전체 접근 설정에서는 기본값이 `live`가 될 수 있다. 일정한 동작이 필요하면 값을 명시한다.

근거: [설정 레퍼런스](https://learn.chatgpt.com/docs/config-file/config-reference).

## 5. 프로젝트 지침과 신뢰

| 키 | 값 | 용도 |
| --- | --- | --- |
| `developer_instructions` | 문자열 | 추가 개발자 지침 |
| `model_instructions_file` | 파일 경로 문자열 | 내장 지침 교체용 고급 옵션. 프로젝트 규칙은 우선 `AGENTS.md`로 관리 권장 |
| `project_doc_max_bytes` | 숫자(바이트) | 프로젝트 지침을 읽는 크기 한도 |
| `project_doc_fallback_filenames` | 문자열 배열 | `AGENTS.md`가 없을 때 찾을 대체 파일명 |
| `projects."<경로>".trust_level` | `"trusted"`, `"untrusted"` | 프로젝트 설정·훅·규칙 로드 여부에 영향 |

사용자 설정 파일의 예시:

```toml
[projects."/absolute/path/to/project"]
trust_level = "trusted"
```

`model_provider`, `model_providers`, `openai_base_url`, `notify`, `otel` 등은 프로젝트 설정에서 무시되므로 사용자 설정에 둔다. `trust_level = "untrusted"`는 유효하지만 `approval_policy = "untrusted"`는 더 이상 지원되지 않는다.

근거: [설정 레퍼런스](https://learn.chatgpt.com/docs/config-file/config-reference), [프로젝트 설정](https://learn.chatgpt.com/docs/config-file/config-advanced#project-config-files-codexconfigtoml).

## 6. MCP 서버 연결

`[mcp_servers.<이름>]` 아래에 설정한다.

| 키 | 값 / 기본값 | 용도 |
| --- | --- | --- |
| `url` | URL 문자열 | HTTP MCP 서버 |
| `command`, `args` | 실행 파일 문자열, 인수 배열 | 로컬 stdio MCP 서버 |
| `enabled` | `true` / `false` | 설정을 보존하면서 사용 여부 변경 |
| `required` | `true` / `false` | `true`이면 활성 서버 초기화 실패 시 시작·재개 실패 |
| `bearer_token_env_var` | 환경변수 이름 | HTTP 인증 토큰을 환경변수에서 읽음 |
| `startup_timeout_sec` | 초 단위 숫자, 기본 `10` | 서버 시작 제한 시간 |
| `tool_timeout_sec` | 초 단위 숫자, 기본 `60` | 도구 호출 제한 시간 |
| `enabled_tools` | 도구 이름 배열 | 노출할 도구 허용 목록 |
| `disabled_tools` | 도구 이름 배열 | 제외 목록. 허용 목록 적용 후 처리 |

공식 문서 MCP 연결 예시:

```toml
[mcp_servers.openaiDeveloperDocs]
url = "https://developers.openai.com/mcp"
enabled = true
startup_timeout_sec = 20
tool_timeout_sec = 60
```

`20`초는 이 예제의 선택값이다. `url` 방식과 `command` 방식은 서버 연결 방식에 맞게 선택한다.

근거: [설정 레퍼런스](https://learn.chatgpt.com/docs/config-file/config-reference), [공식 Docs MCP](https://developers.openai.com/resources/docs-mcp).

## 7. 서브에이전트·스킬·기능 스위치

| 키 | 값 / 기본값 | 용도 |
| --- | --- | --- |
| `agents.enabled` | `true` / `false`, 기본 `true` | 멀티에이전트 도구 활성화 |
| `agents.max_concurrent_threads_per_session` | 숫자 | 기본 에이전트를 제외한 동시 서브에이전트 스레드 상한 |
| `agents.default_subagent_model` | 모델 ID 문자열 | 서브에이전트 기본 모델 |
| `agents.default_subagent_reasoning_effort` | 지원 추론 강도 문자열 | 서브에이전트 기본 추론 강도 |
| `features.apps` | `true` / `false`, 기본 `true` | 앱·커넥터 통합 |
| `features.memories` | `true` / `false`, 기본 `false` | 메모리 기능 |
| `features.hooks` | `true` / `false` | 라이프사이클 훅 |

개별 스킬 비활성화 예시:

```toml
[[skills.config]]
path = "/absolute/path/to/skill-directory"
enabled = false
```

`path`는 `SKILL.md`를 포함하는 **디렉터리**다. 기능 스위치는 해당 기능을 사용할 수 있게 하는 설정이며, 매 작업에서 반드시 실행하도록 지시하는 것은 아니다.

근거: [설정 레퍼런스](https://learn.chatgpt.com/docs/config-file/config-reference).

## 8. 환경변수·기록·인증·알림

| 키 | 옵션 | 의미 |
| --- | --- | --- |
| `shell_environment_policy.inherit` | `"all"`, `"core"`, `"none"` | 하위 명령에 넘기는 환경변수의 기본 범위 |
| `shell_environment_policy.ignore_default_excludes` | `true`(기본) / `false` | `false`이면 이름에 KEY·SECRET·TOKEN이 들어가는 변수의 자동 제외 적용 |
| `shell_environment_policy.filters` | 패턴 → `"include"` / `"exclude"` | 환경변수 필터. 패턴은 대소문자 무시, `*`·`?` 지원 |
| `history.persistence` | `"save-all"`, `"none"` | 로컬 `history.jsonl` 기록 여부 |
| `history.max_bytes` | 숫자(바이트) | 기록 크기 상한. 오래된 항목부터 정리 |
| `cli_auth_credentials_store` | `"file"`, `"keyring"`, `"auto"`, `"ephemeral"` | CLI 인증정보 저장 방식 선택 |
| `tui.notifications` | `true` / `false` 또는 이벤트 문자열 배열 | 터미널 알림 |
| `tui.notification_method` | `"auto"`(기본), `"osc9"`, `"bel"` | 알림 전달 방식 |
| `tui.notification_condition` | `"unfocused"`(기본), `"always"` | 비활성 터미널에서만 알릴지 선택 |
| `tui.alternate_screen` | `"auto"`(기본), `"always"`, `"never"` | 대체 화면 사용 여부. `never`는 스크롤백 유지에 유용 |
| `notify` | 명령·인수 문자열 배열 | 외부 알림 프로그램 실행. 사용자 설정에 작성 |

`history.persistence = "none"`을 모든 로그·서버 데이터 저장의 비활성화로 해석하면 안 된다. 이 항목은 로컬 히스토리 설정이다. 환경변수를 제한하면 기존 빌드·인증 명령이 영향을 받을 수 있다.

근거: [고급 설정](https://learn.chatgpt.com/docs/config-file/config-advanced), [설정 레퍼런스](https://learn.chatgpt.com/docs/config-file/config-reference).

## 9. 복사해서 조정할 시작 예시

아래는 로컬 개발용 제안이다. 모델은 현재 계정에서 사용 가능한 ID로 바꾸며, 기존 파일에는 필요한 항목만 병합한다.

```toml
# ~/.codex/config.toml
#:schema https://developers.openai.com/codex/config-schema.json

model = "gpt-6.1-sol"
model_reasoning_effort = "medium"
approval_policy = "on-request"
approvals_reviewer = "user"
sandbox_mode = "workspace-write"
web_search = "cached"

[sandbox_workspace_write]
network_access = false

[history]
persistence = "save-all"
max_bytes = 104857600 # 100 MiB

[tui]
notifications = true
notification_method = "auto"
notification_condition = "unfocused"
```

작업별 변경분을 별도 프로필로 저장할 수 있다:

```toml
# ~/.codex/research.config.toml
web_search = "live"
```

```sh
codex --profile research
# 아래는 파일을 수정하지 않는 일회성 덮어쓰기
codex -c 'web_search="live"'
```

예제의 구조와 항목 근거: [공식 예제 설정](https://learn.chatgpt.com/docs/config-file/config-sample), [프로필·CLI 덮어쓰기](https://learn.chatgpt.com/docs/config-file/config-advanced#profiles).

## 10. 오래된 예제를 따라 할 때 확인할 변경점

| 오래된 설정 | 현재 문서 기준 |
| --- | --- |
| `approval_policy = "untrusted"` | 지원 종료. `on-request` 또는 `never` 등 지원 정책 선택 |
| `approval_policy = "on-failure"` | 폐기 예정(deprecated). 대화형은 `on-request`, 비대화형은 `never` 검토 |
| `[profiles.<이름>]`, 최상위 `profile = "..."` | **Codex 0.134.0부터 미지원.** `<이름>.config.toml`과 `--profile` 사용 |
| `features.web_search*` | 폐기 예정. 최상위 `web_search` 사용 |
| `experimental_instructions_file` | `model_instructions_file`로 변경 |
| `agents.max_threads` | 기존 별칭. 새 설정은 `agents.max_concurrent_threads_per_session` 사용 |
| 환경변수 `exclude` / `include_only` 배열 | 기존 형식 지원. 새 `filters`와 동일 설정 레이어에서 혼용 금지 |

설정이 적용되지 않으면 설치 버전(`codex --version`), 파일 위치, 프로젝트 신뢰, 상위 우선순위 설정, 조직 정책 순으로 확인한다. CLI 옵션이나 TUI 설정이 데스크톱 앱에서도 동일하게 적용된다고 가정하지 않는다.

근거: [설정 레퍼런스](https://learn.chatgpt.com/docs/config-file/config-reference), [프로필 변경사항](https://learn.chatgpt.com/docs/config-file/config-advanced#profiles).
