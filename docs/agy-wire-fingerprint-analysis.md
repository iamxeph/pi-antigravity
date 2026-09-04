# Google Antigravity (agy CLI) 와이어 지문(Wire Fingerprint) 실측 분석 및 pi-antigravity 보강 가이드

- **작성일자**: 2026-09-04
- **실측 대상**:
  - 순정 Antigravity CLI: `agy` v1.1.23 (CL: 974125021, Linux amd64)
  - 비교 대상 1: `pi-antigravity` (v0.7.1)
  - 비교 대상 2: `CLIProxyAPI` (router-for-me/CLIProxyAPI v7.2.x)
- **도구**: `mitmproxy` / `mitmdump` v12.2.3

---

## 1. 개요 및 목적

Pi Coding Agent에서 `pi-antigravity` 확장을 통해 Antigravity 백엔드(Google Cloud Code Assist)를 사용할 때, 서버 관점에서 **실제 `agy` CLI와 완벽히 구별 불가능한 동일한 네트워크 지문(Wire Fingerprint)**을 생성하도록 소스코드를 보강하기 위한 기술 분석 보고서입니다.

실제 `agy` CLI의 통신 패킷을 `mitmproxy`로 전수 캡처하여 검증한 결과, 현재 `pi-antigravity`는 순정 `agy` CLI가 아닌 **과거 VS Code 확장(Antigravity Hub / Cloud Shell Editor)의 지문**을 전송하고 있으며, 핵심 필드(`thinkingConfig`, 헤더, 라벨 메타데이터 등)에서 명백한 차이가 존재함을 확인했습니다.

---

## 2. 5대 차원별 실측 지문 비교표 (Diff Table)

| 차원 (Dimension)                        | 순정 `agy` CLI (실측)                                                                                                                                                                                                                              | 현재 `pi-antigravity` (v0.7.1)                                                                                                                                                                                   | `CLIProxyAPI`                                                                | 불일치 여부 & 위험도                                                                                                                   |
| :-------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------- |
| **User-Agent**                          | `antigravity/cli/1.1.23 (aidev_client; os_type=linux; arch=amd64; cl=974125021; auth_method=consumer)`                                                                                                                                             | `antigravity/hub/2.8.0 (aidev_client; os_type=darwin; arch=arm64; cl=963137146)`                                                                                                                                 | `antigravity/hub/2.11.0 darwin/arm64` (매니페스트 자동 갱신)                 | **해결 완료 (PR 1)**<br>기존: `cli` 대신 `hub` 표기, Linux amd64 환경에서도 `darwin/arm64` 고정, `auth_method=consumer` 누락           |
| **HTTP Headers**                        | - `Host`<br>- `User-Agent`<br>- `Authorization: Bearer ...`<br>- `Content-Type: application/json`<br>- `Accept-Encoding: gzip`                                                                                                                     | - 상동 헤더 포함<br>- **추가 전송 (누출)**:<br> * `Accept: text/event-stream`<br> * `X-Goog-Api-Client: google-cloud-sdk vscode_cloudshelleditor/0.1`<br> * `Client-Metadata: {"ideType":"ANTIGRAVITY",...}`     | - 헤더 정돈 완료 (VS Code 헤더 없음)<br>- `Accept: text/event-stream` 미전송 | **해결 완료 (PR 1)**<br>기존: VS Code 플러그인 전용 헤더 누출로 비공식 클라이언트 즉시 식별 가능                                       |
| **Thinking Config**                     | `thinkingConfig: { includeThoughts: true, thinkingBudget: <int> }`<br>- Gemini 3.8/3.7/3.6: High `-1`, Med `4000`, Low `1000`, Off `0`<br>- Gemini 3.1 Pro: High `10001`, Low `1001`, Off `0`<br>- Claude Sonnet/Opus: `1024`<br>- GPT-OSS: `8192` | `thinkingConfig: { includeThoughts: true, thinkingLevel: "HIGH" }`<br>(Claude, GPT-OSS는 `thinkingConfig` 생략)                                                                                                  | `auto`면 `-1`, 그 외 `thinkingLevel` 문자열                                  | **해결 완료 (PR 2)**<br>기존: 전 모델에서 `thinkingLevel` 문자열 대신 정수 예산(`thinkingBudget`) 전송 필요                            |
| **Labels (`request.labels`)**           | ```json                                                                                                                                                                                                                                            |
| {                                       |
| "last_step_index": "0",                 |
| "model_enum": "MODEL_PLACEHOLDER_M298", |
| "request_id": "<traj_id>-0",            |
| "trajectory_id": "<uuid>",              |
| "used_claude": "false",                 |
| "used_claude_conservative": "false",    |
| "used_non_gemini_model": "false"        |
| }                                       |
| ```                                     | ```json                                                                                                                                                                                                                                            |
| {                                       |
| "last_step_index": "1",                 |
| "trajectory_id": "<uuid>",              |
| "used_claude": "false",                 |
| "used_claude_conservative": "false"     |
| }                                       |
| ```                                     | 미구현 또는 단순 래핑                                                                                                                                                                                                                              | **불일치 (높음 - PR 4 예정)**<br>- `request_id` 누락<br>- `used_non_gemini_model` 누락<br>- `model_enum` 동적 매핑 실패 (구형 3.5 모델만 정적 하드코딩되어 3.7/3.8에서 증발)<br>- `last_step_index`가 '1'로 고정 |
| **Request ID**                          | `agent/<conv_uuid>/<epoch_ms>/<traj_uuid>/<step>`                                                                                                                                                                                                  | `agent/<new_uuid>/<epoch_ms>/<traj_uuid>/2`                                                                                                                                                                      | `agent-<uuid>`                                                               | **불일치 (낮음 - PR 4 예정)**<br>`pi-antigravity`는 세션 유지 대신 턴마다 conv_uuid를 재생성하고 step을 2로 고정                       |
| **Tool Config**                         | Gemini / Claude / GPT-OSS 전 모델 호출 시 `toolConfig` 생략 (`undefined`)                                                                                                                                                                          | `toolConfig: { functionCallingConfig: { mode: "VALIDATED" } }` 항상 강제 전송                                                                                                                                    | Claude 등 특정 모델만 `VALIDATED` 적용                                       | **해결 완료 (PR 3)**<br>기존: 도구 존재 시 무조건 `mode: "VALIDATED"`를 강제하고, 도구 없는 Claude에도 주입하여 VS Code 확장 지문 누출 |
| **Endpoint Priority**                   | `https://daily-cloudcode-pa.googleapis.com` 단일 호출                                                                                                                                                                                              | daily $\rightarrow$ sandbox $\rightarrow$ prod(cloudcode-pa) 순차 폴백                                                                                                                                           | daily $\rightarrow$ prod 폴백                                                | **양호**<br>daily 1순위는 일치                                                                                                         |
| **loadCodeAssist Payload**              | `{"metadata":{"ideType":"ANTIGRAVITY"}}`                                                                                                                                                                                                           | `{"metadata":{"ideType":"ANTIGRAVITY","platform":"PLATFORM_UNSPECIFIED","pluginType":"GEMINI"}}`                                                                                                                 | `{"metadata":{"ideType":"ANTIGRAVITY"}}`                                     | **해결 완료 (PR 1)**<br>기존: 불필요한 metadata 필드 포함                                                                              |
| **OAuth Flow**                          | Client ID 및 Scope 6종 일치                                                                                                                                                                                                                        | Client ID 및 Scope 6종 일치                                                                                                                                                                                      | Client ID 및 Scope 6종 일치                                                  | **일치**                                                                                                                               |

---

## 3. 상세 분석 및 근거 (Deep-Dive)

### 3.1. User-Agent 포맷

- **순정 agy CLI 형식**:
  ```text
  antigravity/cli/<version> (aidev_client; os_type=<os>; arch=<arch>; cl=<cl>; auth_method=consumer)
  ```
  - `os_type`: Node의 `process.platform` 매핑 (`linux` $\rightarrow$ `linux`, `darwin` $\rightarrow$ `darwin`, `win32` $\rightarrow$ `windows`)
  - `arch`: Node의 `process.arch` 매핑 (`x64` $\rightarrow$ `amd64`, `arm64` $\rightarrow$ `arm64`)
  - `cl`: agy 빌드 체인지리스트 번호 (1.1.23 기준 `974125021`)
  - `auth_method`: 개인 계정 로그인 시 `consumer`
- **구버전 `pi-antigravity` 문제점**:
  - `antigravity/hub/2.8.0`을 사용하며, OS와 무관하게 `os_type=darwin; arch=arm64`로 하드코딩되어 있고 `auth_method=consumer`가 빠져 있었습니다.
- **개선 반영 (PR 1 완료)**:
  - `defaultUserAgent()`를 순정 `agy` 1.1.23 Linux amd64 실측 문자열로 고정하고, `ANTIGRAVITY_USER_AGENT` 환경변수로 사용자 정의를 완벽히 지원하도록 정돈했습니다 (`da9514b`, `811e464`).

### 3.2. HTTP Header 누출

- `src/client/client.ts`의 구 `antigravityHeaders` 함수가 다음 헤더들을 강제 주입하고 있었습니다:
  - `X-Goog-Api-Client: google-cloud-sdk vscode_cloudshelleditor/0.1`
  - `Client-Metadata: {"ideType":"ANTIGRAVITY","platform":"LINUX","pluginType":"GEMINI"}`
  - `Accept: text/event-stream`
- **실측 증거**:
  `agy` CLI가 보낸 모든 HTTP 요청(`streamGenerateContent`, `fetchAvailableModels`, `retrieveUserQuotaSummary` 등)에서 위의 3개 헤더는 단 한 번도 나타나지 않았습니다. `agy`는 `Host`, `User-Agent`, `Authorization`, `Content-Type`, `Accept-Encoding: gzip`만 전송합니다.
- **개선 반영 (PR 1 완료)**:
  - 비표준 헤더 3종을 전면 제거하고 불필요한 `loadCodeAssist` 페이로드 필드를 축소했습니다 (`da9514b`).

### 3.3. `model_enum` 동적 매핑 결여

- `agy` CLI는 요청 시 `request.labels.model_enum`에 `MODEL_PLACEHOLDER_M298`과 같은 내부 enum 문자열을 넣습니다.
- 이 enum 값은 별도의 정적 테이블이 아니라, `/v1internal:fetchAvailableModels` 응답의 `data.models[modelId].model` 필드에 정확히 들어있습니다.
  - `gemini-3.7-flash-high` $\rightarrow$ `"model": "MODEL_PLACEHOLDER_M298"`
  - `gemini-3.7-flash-medium` $\rightarrow$ `"model": "MODEL_PLACEHOLDER_M299"`
  - `gemini-3.8-flash-high` $\rightarrow$ `"model": "MODEL_PLACEHOLDER_M318"`
  - `claude-sonnet-4-6` $\rightarrow$ `"model": "MODEL_PLACEHOLDER_M35"`
- 현재 `pi-antigravity`는 `models.ts`에 구형 3.5 모델 5개만 정적으로 하드코딩해 두었기 때문에, 3.7이나 3.8 등 최신 모델을 쓸 때 `model_enum` 라벨이 아예 누락됩니다.

### 3.4. Thinking Configuration (`thinkingBudget`)

- 순정 `agy` 1.1.23 전수 실측 결과, 14개 전체 모델에서 `thinkingLevel` ("HIGH" 등) 문자열 enum은 아예 전송되지 않으며, 메타데이터에 정의된 정수 예산(`thinkingBudget`)을 전송합니다:
  - **Gemini 3.8 / 3.7 / 3.6 Flash**: High `-1`, Medium `4000`, Low `1000`, Off `0`
  - **Gemini 3.1 Pro**: High `10001` (런타임 `gemini-pro-agent`), Low `1001`, Off `0`
  - **Claude Sonnet 4.6 / Opus 4.6 Thinking**: `1024`, Off `0`
  - **GPT-OSS 120B**: `8192`, Off `0`
- `pi-antigravity` 기존 소스코드는 Gemini 3.x Flash에 대해 `thinkingLevel: "HIGH"` 문자열을 보내고, Claude/GPT-OSS는 `thinkingConfig`를 생략하고 있어 순정과 불일치했습니다.
- **개선 반영 (PR 2 완료)**:
  - `ThinkingWire` 타입을 정의하고 Gemini(3.8/3.7/3.6/3.5), Claude, GPT-OSS 전 모델에 대해 순정 CLI와 동일한 정수 `thinkingBudget` 예산을 반환하도록 정규화했습니다 (`8d8ff77`).

---

## 4. `pi-antigravity` 소스코드 보강 청사진 및 단계별 PR 로드맵

수정 대상 리포지토리: `~/Projects/pi-antigravity`

독립적인 검증 및 안전한 업스트림 머지를 위해 총 4단계의 PR로 분할하여 진행합니다.

---

### [완료] PR 1: 헤더 및 User-Agent 정돈 (`fix/wire-headers`)

- **PR**: [Rahularya01/pi-antigravity#35](https://github.com/Rahularya01/pi-antigravity/pull/35)
- **반영 브랜치 / 커밋**: `fix/wire-headers` (`da9514b`, `811e464`)
- **주요 반영 내용**:
  1. **User-Agent 순정 CLI 고정 (`src/client/client.ts`)**:
     - `defaultUserAgent()`를 순정 `agy` 1.1.23 실측 문자열로 고정:
       `antigravity/cli/1.1.23 (aidev_client; os_type=linux; arch=amd64; cl=974125021; auth_method=consumer)`
     - 미문서화된 레거시 `HUB_*` 관련 내부 환경변수 제거 (`ANTIGRAVITY_USER_AGENT` 하나로 전체 오버라이드 통일)
  2. **HTTP Headers 누출 제거 (`src/client/client.ts`)**:
     - `antigravityHeaders()`에서 `X-Goog-Api-Client`, `Client-Metadata`, `Accept: text/event-stream` 비표준 헤더 3종 전면 제거
     - `Host`, `User-Agent`, `Authorization`, `Content-Type`, `Accept-Encoding: gzip`만 전송되도록 정돈
  3. **`loadCodeAssist` 페이로드 축소 (`src/client/client.ts`)**:
     - 바디를 `{"metadata":{"ideType":"ANTIGRAVITY"}}`로 축소 (불필요한 `platform`, `pluginType` 제거)
  4. **Dead Code 제거 (`src/types/enums.ts`)**:
     - `Client-Metadata` 제거로 참조가 0개가 된 `Platform` enum 삭제
  5. **단위 테스트 추가 (`scripts/test-model-routing.ts`)**:
     - User-Agent 문자열 완전 일치 및 비표준 헤더 부재(`undefined`) 검증

---

### [완료] PR 2: 전 모델 Thinking Budget 정수 예산화 (`fix/thinking-budget`)

- **반영 브랜치 / 커밋**: `fix/thinking-budget` (`8d8ff77`), `main`에 병합 완료 (`ea6896a`)
- **배경**:
  - 순정 `agy` CLI 실측 결과, Gemini, Claude, GPT-OSS를 포함한 14개 전 모델에서 `thinkingLevel` 문자열 enum은 절대 전송되지 않으며, 메타데이터에 정의된 정수 예산(`thinkingBudget`)을 전송함
  - 기존 소스코드는 Gemini 3.x Flash에 대해 `thinkingLevel` 문자열을 보내고, Claude와 GPT-OSS는 `thinkingConfig`를 생략하여 순정과 불일치
- **주요 반영 내용**:
  1. **타입 정의 및 정돈 (`src/types/types.ts`)**:
     - `ThinkingWire` 타입 정의 (`{ includeThoughts: boolean; thinkingBudget: number; }`)
     - `GeminiGenerationConfig.thinkingConfig`에 `ThinkingWire` 적용 및 미사용 문자열 enum(`GeminiThinkingLevel`) 제거
  2. **`thinkingBudget` 전 모델 정규화 (`src/models/models.ts`)**:
     - `getThinkingConfig`에서 전 모델 대상 정수 예산(`thinkingBudget`) 반환:
       - Gemini 3.8/3.7/3.6 Flash: High/XHigh `-1`, Medium `4000`, Low/Minimal `1000`, Off `0`
       - Gemini 3.1 Pro / 3.5 Flash: High `10000`, Medium `4000`, Low `1000`, Off `0`
       - Claude Sonnet/Opus Thinking: `1024`, Off `0`
       - GPT-OSS 120B: `8192`, Off `0`
     - 불필요한 `googleLevel()` 헬퍼 함수 삭제 및 `ThinkingWire` re-export
  3. **문서 및 스모크 테스트 페이로드 갱신**:
     - `README.md`: Antigravity 공식 CLI 와이어 규격에 맞춰 전 모델 정수 `thinkingBudget` 전송 사실 명시
     - `scripts/smoke-all-models.mjs`: `thinkingBudget` 규격(-1 / 4000 / 1000 / 0)에 맞춰 페이로드 생성 로직 갱신
  4. **단위 및 라우팅 회귀 테스트 갱신**:
     - `scripts/test-model-discovery.ts`: Gemini 전 계열, Claude (1024 / 0), GPT-OSS (8192 / 0)의 `thinkingBudget` assertion 추가 및 갱신
     - `scripts/test-model-routing.ts`: Gemini 3.8/3.7/3.6/3.5, Claude Sonnet, GPT-OSS의 `buildRequest` 시 `thinkingBudget` 및 `includeThoughts` 정합성 검증 추가

---

### [완료] PR 3: 전 모델 기본 호출 시 `toolConfig` 생략 (`fix/gemini-tool-config`)

- **PR**: [Rahularya01/pi-antigravity#38](https://github.com/Rahularya01/pi-antigravity/pull/38)
- **반영 브랜치 / 커밋**: `fix/gemini-tool-config` (`079c7df`)
- **배경**:
  - 순정 `agy` CLI 실측 결과, Gemini뿐만 아니라 Claude, GPT-OSS를 포함한 14개 전 모델 호출 시 17개 도구가 선언되어 있어도 기본 Auto 모드에서는 `request.toolConfig` 필드가 아예 생략됨 (`undefined`)
  - 기존 `pi-antigravity`는 도구가 존재할 때 무조건 `request.toolConfig = { functionCallingConfig: { mode: "VALIDATED" } }`를 강제 주입하고, 도구가 없는 Claude에도 강제 주입 중이었음
- **주요 반영 내용**:
  1. **`src/stream/stream.ts` (`buildRequest`)**:
     - 기본 모드(Auto / 미지정): 전 모델에서 `request.toolConfig` 필드 자체를 생략 (`undefined`)
     - 명시적 `toolChoice`(`none`, `any`, `required`): 지정된 경우에만 `toolConfig` 설정 (`NONE` / `ANY`)
     - 도구 없는 Claude에 강제 주입되던 레거시 `else if (isClaude)` 블록 완전 제거
  2. **타입 호환성 정합 (`src/types/types.ts`)**:
     - `AntigravityStreamOptions.toolChoice`를 `${ToolChoice}`로 확장하여 Pi `SimpleStreamOptions` 문자열(`"auto" | "none"`)과의 완벽한 호환성 확보 및 `src/types/enums.ts`의 표준 enum 형태 유지
  3. **단위 및 회귀 테스트 갱신 (`scripts/test-model-routing.ts`)**:
     - Gemini, Claude, GPT-OSS 모델별 기본 호출 시 `toolConfig`가 `undefined`인지 검증
     - 명시적 `toolChoice`(`auto`, `none`, `any`, `required`) 및 문자열 리터럴(`"auto"`, `"none"`) 호환성 검증 테스트 8종 추가

---

### [대기] PR 4: Request Envelope 라벨 정규화 및 `model_enum` 동적 캐시 (`feat/envelope-labels`)

- **브랜치명**: `feat/envelope-labels` (Worktree: `~/Projects/pi-antigravity-envelope-labels`)
- **배경**:
  - 순정 `agy` CLI 실측 라벨:
    ```json
    {
      "last_step_index": "0",
      "model_enum": "MODEL_PLACEHOLDER_M298",
      "request_id": "<trajectory_id>-0",
      "trajectory_id": "<uuid>",
      "used_claude": "false",
      "used_claude_conservative": "false",
      "used_non_gemini_model": "false"
    }
    ```
  - 현재 `pi-antigravity`는 `request_id`, `used_non_gemini_model`이 누락되어 있고, `last_step_index`가 `"1"`로 고정되어 있으며, 구형 3.5 모델 외 최신 모델의 `model_enum`이 증발함
- **변경 상세**:
  1. **`src/types/types.ts` (`ModelInfoRaw`)**:
     - `model?: unknown` 필드 추가 (`MODEL_PLACEHOLDER_M...` enum 파싱용)
  2. **동적 `model_enum` 추출 및 캐시 (`src/client/client.ts`, `src/models/discovery.ts`)**:
     - `/v1internal:fetchAvailableModels` 응답의 `data.models[modelId].model` 값을 추출하여 런타임 맵/캐시에 보관
  3. **Envelope 및 Labels 정규화 (`src/utils/util.ts`)**:
     - `labels.last_step_index`: `${step - 1}` (첫 턴 `"0"`)
     - `labels.request_id`: `${trajectoryId}-${step - 1}` 추가
     - `labels.used_non_gemini_model`: 비-Gemini(Claude, GPT-OSS 등) 여부에 따라 `"true"` / `"false"` 추가
     - `labels.model_enum`: 런타임 캐시에서 조회하여 주입 (기존 하드코딩 맵은 디스커버리 전 fallback으로만 활용)
     - `requestId`: `agent/${convId}/${Date.now()}/${trajectoryId}/${step}` 포맷 유지 및 턴 카운터(step) 지원
  4. **테스트 코드 갱신**:
     - `scripts/test-model-discovery.ts`, `scripts/test-model-routing.ts`에 동적 `model_enum` 추출 및 신규 라벨 필드 검증 추가

---

## 5. 재현 및 검증 레시피 (mitmproxy Capture Recipe)

향후 `agy` 업데이트 시 재검증을 위한 표준 절차:

1. **mitm 덤프 스크립트 작성 (`/tmp/agy-capture/addon.py`)**:

   ```python
   import json, gzip
   from mitmproxy import http

   def _body(c):
       if not c: return ""
       try: return gzip.decompress(c).decode("utf-8", errors="replace")
       except: pass
       try: return c.decode("utf-8", errors="replace")
       except: return f"<binary {len(c)} bytes>"

   def response(flow: http.HTTPFlow) -> None:
       with open("/tmp/agy-capture/flows.jsonl", "a", encoding="utf-8") as f:
           f.write(json.dumps({
               "req": {"method": flow.request.method, "url": flow.request.url, "headers": dict(flow.request.headers), "body": _body(flow.request.raw_content)},
               "resp": {"status": flow.response.status_code if flow.response else None, "headers": dict(flow.response.headers) if flow.response else {}, "body": _body(flow.response.raw_content) if flow.response else None}
           }, ensure_ascii=False) + "\n")
   ```

2. **프록시 실행**:

   ```bash
   nix-shell -p mitmproxy --run "mitmdump -p 18080 -s /tmp/agy-capture/addon.py"
   ```

3. **애플리케이션 요청 실행**:

   ```bash
   HTTPS_PROXY=http://127.0.0.1:18080 HTTP_PROXY=http://127.0.0.1:18080 SSL_CERT_FILE=~/.mitmproxy/mitmproxy-ca-cert.pem agy -p "say pong" --model gemini-3.7-flash-high
   ```

4. **패킷 대조 확인**:
   ```bash
   jq -c 'select(.req.url | contains("streamGenerateContent")) | {headers: .req.headers, body: (.req.body | fromjson)}' /tmp/agy-capture/flows.jsonl
   ```
