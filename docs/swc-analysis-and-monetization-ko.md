# SWC 저장소 분석 및 활용 가이드 (한국어)

이 문서는 SWC 저장소를 전수조사한 결과와, 이를 학습·서비스·수익화에 활용하는 방법을
한국어로 정리한 기록이다.

- 작성일: 2026-10-08
- 분석 대상 저장소(포크): https://github.com/bmshin94/swc
- 원본(업스트림) 저장소: https://github.com/swc-project/swc
- 공식 문서: https://swc.rs/docs/installation/
- Rust API 문서: https://rustdoc.swc.rs/swc/
- 라이선스: Apache-2.0 (상업적 이용·수정·재배포 가능, 저작권 고지 유지 필요)

---

## 1. 이 저장소는 무엇인가

**SWC (Speedy Web Compiler)** — Rust로 작성된 초고속 TypeScript / JavaScript 컴파일러.
Rust 라이브러리와 JavaScript 라이브러리를 동시에 제공한다.

| 항목 | 값 |
|---|---|
| 프로젝트 | SWC (Speedy Web Compiler) |
| 언어 | Rust (+ Node.js / WASM 바인딩) |
| 크레이트 수 | 약 120개 (`crates/`) |
| 코드 규모 | Rust 파일 약 1,400개 / 약 80만 줄 |
| 패키지 버전 | `@swc/core` 1.16.8-nightly-20260918.1 |
| 작성자 | 강동윤 (kdy1997.dev@gmail.com) |
| MSRV | Rust 1.73 |
| Node | 사용 v10+, 개발 v20+ / pnpm 10.33+ |

### 대체하는 도구

| 역할 | 기존 도구 | SWC |
|---|---|---|
| TypeScript → JavaScript | `tsc`, `babel` | 지원 |
| JSX/TSX 변환 | `@babel/preset-react` | 지원 |
| 구형 브라우저 호환 변환 | `@babel/preset-env` | 지원 |
| 코드 압축 (minify) | `terser`, `uglify-js` | 지원 |
| 모듈 변환 (ESM/CJS/AMD/UMD) | `babel`, `rollup` | 지원 |
| CSS 파싱/압축/프리픽스 | `postcss`, `autoprefixer` | 지원 |
| HTML / XML 파싱·압축 | `html-minifier` | 지원 |
| 번들링 | `webpack`, `rollup` | 실험적 지원 |

### 실제 채택 사례

Next.js(Vercel), Deno, Parcel, Rspack, `@swc/jest`, Vite 생태계 등이 내부 엔진으로 사용한다.

### 왜 빠른가

1. **네이티브 실행** — Babel은 JavaScript, SWC는 Rust(기계어)로 동작.
2. **병렬 처리** — 파일 단위로 멀티코어를 전부 활용 (JS는 기본 싱글스레드).
3. **메모리 최적화** — `swc_atoms`(문자열 인터닝), `swc_allocator`(아레나 할당),
   `swc_malloc`(mimalloc). 프로젝트 규칙 자체가 "항상 성능 우선"이다.

---

## 2. 컴파일 파이프라인

```
소스 코드 (TS / TSX / 최신 JS)
  ↓ swc_ecma_lexer      : 문자열을 토큰으로 분해
  ↓ swc_ecma_parser     : 토큰을 AST로 조립
  ↓ swc_ecma_transforms_*: AST 변환 (타입 제거, JSX, 문법 다운그레이드, 최적화)
  ↓ swc_ecma_minifier   : 압축 / 변수명 단축
  ↓ swc_ecma_codegen    : AST를 다시 코드 문자열로 출력
결과 JavaScript (+ 소스맵)
```

### 변환 예시

입력:

```tsx
const Hello = ({ name }: { name: string }) => <h1>안녕 {name}!</h1>;
```

출력:

```js
const Hello = ({ name }) => /*#__PURE__*/ React.createElement("h1", null, "안녕 ", name, "!");
```

- 타입 주석 제거 → `swc_ecma_transforms_typescript`
- JSX → 함수 호출 → `swc_ecma_transforms_react`
- 화살표 함수 다운그레이드(옵션) → `swc_ecma_compat_es2015`

---

## 3. 폴더 전수조사

```
swc/
├── crates/        Rust 소스 (약 120 크레이트) — 엔진 본체
├── packages/      npm 배포 패키지 (core, minifier, html, react-compiler, types, helpers)
├── bindings/      Node / WASM / CLI 바인딩
├── docs/          설계 문서 및 ADR
├── scripts/       벤치·배포·테스트 자동화 스크립트
├── tools/, xtask/ 코드 생성, 릴리스, 네이티브 애드온 패킹
├── rules/         ast-grep 린트 규칙 (예: `.into().into()` → `.into()`)
├── .agents/       AI 에이전트 스킬 4종
├── .github/       워크플로우 15종, 봇, 생태계 CI
├── AGENTS.md      AI 에이전트 공통 규칙
├── CLAUDE.md      규칙 색인 + 페르소나 가이드
├── ARCHITECTURE.md 내부 구조 개요
└── .mcp.json      MCP 서버 설정 (CodSpeed)
```

### 3.1 `crates/` 주요 그룹

- **기반**: `swc_common`(span/hygiene/에러), `swc_atoms`, `swc_allocator`, `swc_arena`,
  `swc_sourcemap`, `swc_config`
- **파싱**: `swc_ecma_parser`, `swc_ecma_lexer`, `swc_ecma_ast`, `swc_ecma_visit`,
  `swc_ecma_utils`, `swc_ecma_quote`
- **변환**: `swc_ecma_transforms_base`(resolver / hygiene / fixer),
  `_typescript`, `_react`, `_module`, `_optimization`, `_proposal`, `_classes`, `_compat`,
  `swc_ecma_compat_es3` ~ `es2022`, `swc_ecma_compat_bugfixes`, `swc_ecma_compat_regexp`
- **압축·출력**: `swc_ecma_minifier`, `swc_ecma_codegen`
- **CSS**: `swc_css_parser`, `_codegen`, `_minifier`, `_prefixer`, `_modules`, `_lints`, `_compat`
- **HTML / XML**: `swc_html_*`, `swc_xml_*`
- **TypeScript 전용**: `swc_ts_fast_strip`(타입만 초고속 제거), `swc_typescript`
- **플러그인 시스템**: `swc_plugin`, `swc_plugin_runner`, `swc_plugin_proxy`,
  `swc_plugin_backend_wasmer`, `swc_plugin_backend_wasmtime`, `swc_plugin_macro`
- **React Compiler**: `swc_ecma_react_compiler`
- **번들러**: `swc_bundler`, `swc_node_bundler`, `swc_graph_analyzer`
- **도구/테스트**: `swc_cli_impl`, `swc-ast-explorer`, `dbg-swc`, `jsdoc`,
  `testing`, `testing_macros`, `swc_ecma_testing`

### 3.2 `packages/core` 공개 API

`packages/core/src/index.ts` 기준: `transform`, `transformSync`, `transformFile`,
`transformFileSync`, `parse`, `parseSync`, `parseFile`, `print`, `printSync`,
`minify`, `minifySync`, `bundle`, `plugins`, `experimental_analyze`, `getBinaryMetadata`.

### 3.3 `bindings/`

- `binding_core_node` — napi-rs 기반 Node 네이티브 애드온 (12개 타겟 빌드)
- `binding_core_wasm`, `binding_minifier_wasm`, `binding_typescript_wasm` — 브라우저 실행용
- `binding_es_ast_viewer` — AST 뷰어
- `binding_native_addon`, `swc_cli` — 애드온 캐리어, CLI 바이너리

### 3.4 AI 에이전트 관련 설정 (이 저장소의 특징)

| 파일 / 폴더 | 역할 |
|---|---|
| `AGENTS.md` (루트) | 전체 규칙: 코드 스타일, 커밋, 픽스처 테스트, 검증 절차 |
| `crates/*/AGENTS.md` | 크레이트별 픽스처 경로와 테스트 명령 |
| `CLAUDE.md` | 위 규칙을 디렉토리별로 합친 색인 + 페르소나 가이드 |
| `.agents/skills/` | `add-changeset`, `repair-pr`, `codspeed-optimize`, `codspeed-setup-harness` |
| `.mcp.json` | MCP 서버: CodSpeed (`https://mcp.codspeed.io/mcp`) |
| `.claude/settings.json`, `.codex/config.toml` | 에이전트 도구 설정 |
| `.github/workflows/claude.yml` | 이슈/PR에 `@claude` 멘션 시 자동 작업 (OWNER/MEMBER/COLLABORATOR만 트리거) |

배울 점:
1. 계층형 규칙 파일(루트 + 디렉토리별)로 작업 위치에 맞는 지침 적용
2. 재사용 스킬로 반복 작업 표준화
3. "`cargo fmt` → `cargo clippy -D warnings` → `cargo test -p <crate>`" 검증 게이트 명시
4. 워크플로우 권한 최소화(`permissions: {}`) + 작성자 권한 체크로 보안 확보
5. 픽스처/스냅샷 테스트 중심 → 에이전트가 스스로 결과 검증 가능

---

## 4. 설치 및 사용법

### 4.1 라이브러리로 사용 (가장 빠른 길)

```bash
npm i -D @swc/core @swc/cli
npx swc src/index.ts -o dist/index.js   # 단일 파일
npx swc src -d dist --watch             # 폴더 + 감시
```

`.swcrc`:

```json
{
  "jsc": {
    "parser": { "syntax": "typescript", "tsx": true },
    "target": "es2020",
    "transform": { "react": { "runtime": "automatic" } },
    "minify": { "compress": true, "mangle": true }
  },
  "module": { "type": "es6" },
  "sourceMaps": true
}
```

```js
import { transform, minify } from "@swc/core";
const { code, map } = await transform(src, { filename: "a.tsx" });
```

연동 패키지: `@swc/jest`(테스트 가속), `swc-loader`(webpack),
`rollup-plugin-swc3`(rollup), `@swc-node/register`(Node 직접 실행).

### 4.2 이 저장소를 직접 빌드

```bash
pnpm install
cargo build                       # 전체 빌드 (초회 10~30분)
cd packages/core && pnpm build:dev && pnpm test
```

- 크레이트 테스트: `cargo test -p swc_ecma_minifier`
- 픽스처 갱신: `UPDATE=1 cargo test -p swc_ecma_minifier`
- 미니파이어 실행 테스트: `crates/swc_ecma_minifier/scripts/exec.sh`
- 종료 전 필수 검증: `cargo fmt --all` → `cargo clippy --all --all-targets -- -D warnings`

### 4.3 Rust 라이브러리로 사용

```toml
[dependencies]
swc_ecma_parser = "*"
swc_ecma_ast    = "*"
swc_ecma_codegen = "*"
```

필요한 크레이트만 골라 쓸 수 있도록 설계되어 있다(린터는 파서만, 번들러는 파서+코드젠).

---

## 5. 자주 묻는 질문 정리

### 플러그인인가, 스킬인가, MCP인가

SWC 본체는 **컴파일러/라이브러리**다. 다만 이 저장소 안에 세 가지가 함께 존재한다.

| 개념 | 위치 | 설명 |
|---|---|---|
| 플러그인 | `crates/swc_plugin*` | SWC는 **플러그인 호스트**. Rust로 작성해 WASM으로 빌드해 꽂는다 |
| 스킬 | `.agents/skills/` | Claude Code 작업 레시피 (SWC 기능이 아니라 개발 보조) |
| MCP | `.mcp.json` | CodSpeed 벤치마크 MCP 서버 (개발 보조) |

반대로 SWC를 감싸 "코드 변환 MCP 서버"를 만드는 것은 가능하며, 유력한 수익화 아이디어다.

### API 토큰이 필요한가

SWC 사용 자체는 **토큰 불필요, 전부 로컬 실행, 네트워크 없음**.
토큰은 주변 작업에만 필요하다.

| 용도 | 토큰 | 필수 여부 |
|---|---|---|
| 설치·사용·빌드·테스트 | 없음 | — |
| GitHub PR 기여 | GitHub 인증 | 기여 시 |
| `claude.yml` 자동화 | `ANTHROPIC_API_KEY` (레포 Secret) | AI 자동화 시 |
| CodSpeed MCP | CodSpeed 토큰 | 벤치 시 |
| npm / crates.io 배포 | npm, cargo 토큰 | 배포 시 |

### AI 에이전트 구축에 도움이 되는가

**엔진 측면**
- 거대 레포를 AST로 파싱해 필요한 범위만 추출 → LLM 토큰 대폭 절감
- AI 생성 코드의 문법 검증(환각 차단)
- 코드 그래프 기반 정밀 RAG
- AST 레벨 안전 리팩터링(문자열 치환보다 안전)
- WASM 빌드가 있어 브라우저/엣지에서도 코드 분석 가능

**운영 측면**
- 계층형 `AGENTS.md`, 재사용 스킬, 검증 게이트, 최소 권한 워크플로우 등
  대규모 코드베이스에 에이전트를 붙이는 모범 사례를 그대로 참고할 수 있다.

### React나 PHP로 만들 수 있는가

- **React**: 가능. `@swc/wasm-web`를 브라우저에서 로드해 플레이그라운드, AST 뷰어,
  실시간 변환기 등을 만들 수 있다. 서버 비용이 들지 않는다.
  저장소의 `bindings/binding_core_wasm`, `binding_es_ast_viewer`가 참고 자료다.
- **PHP**: SWC CLI를 `exec()`로 호출하거나 Node 사이드카/HTTP 마이크로서비스로 감싸 사용 가능.
  PHP로 컴파일러를 직접 구현하는 것은 성능상 비현실적이다.
- **재작성**: SWC 자체를 JS/PHP로 재작성하면 존재 이유(성능)가 사라진다.
  권장 조합은 `React 프론트 + WASM(SWC)` 또는 `Node API 서버 + 필요 시 PHP 백오피스`.

### 유튜브 강의 제작이 가능한가

가능하다. Apache-2.0이므로 출처만 밝히면 문제없다.
한국어 컴파일러 콘텐츠가 희소하고, 창시자가 한국인이라는 스토리도 활용할 수 있다.

| 회차 | 주제 |
|---|---|
| 1 | Babel → SWC 전환 빌드 속도 실측 |
| 2 | `.swcrc` 옵션 완전 정복 |
| 3 | `@swc/jest`로 테스트 가속 |
| 4 | 컴파일러 내부 1 — 렉서·파서·AST |
| 5 | 컴파일러 내부 2 — 변환 패스 추적 |
| 6 | WASM 플러그인 직접 만들기 |
| 7 | 브라우저에서 돌리는 SWC 플레이그라운드 (React) |
| 8 | AI 에이전트 + AST로 토큰 절감 |
| 9 | 오픈소스 첫 PR 보내기 |
| 10 | SWC의 AI 설정 전부 분석 (AGENTS.md / 스킬 / MCP) |

제작 팁: 1화에 성능 실측 배치, 긴 빌드는 캐시 후 녹화, 벤치는 `scripts/bench`·CodSpeed 활용,
Shorts로 짧은 클립 양산.

---

## 6. 수익화 아이디어 10선

| # | 아이디어 | 난이도 | 예상 수익 | 기간 |
|---|---|---|---|---|
| 1 | 빌드 가속 컨설팅 / 마이그레이션 대행 | ★★ | 건당 300만~2,000만원 | 즉시 |
| 2 | 유료 SWC 플러그인 (WASM) | ★★★★ | $29~299/년 | 1~3개월 |
| 3 | 코드 변환 SaaS / API | ★★★ | 월 $19~499 | 2~3개월 |
| 4 | AI 코드 분석 엔진 / MCP 서버 | ★★★ | 월 $29~999 또는 종량 | 1~2개월 |
| 5 | 교육 콘텐츠 (유튜브·강의·전자책) | ★★ | 월 50만~1,000만원 | 3~6개월 |
| 6 | 온라인 플레이그라운드 / 개발자 툴 | ★★ | 월 30만~300만원 | 2~4주 |
| 7 | 사내 빌드 플랫폼 (B2B) | ★★★★ | 연 1,000만~1억 | 3~6개월 |
| 8 | npm 패키지 생태계 + 스폰서십 | ★★ | 간접(평판·단가 상승) | 지속 |
| 9 | 코드 품질·보안 스캐너 | ★★★★ | 월 $49~999 | 3~4개월 |
| 10 | 레거시 현대화 전문 회사 | ★★★★★ | 프로젝트당 수천만~수억 | 6개월+ |

### 상세

1. **빌드 가속 컨설팅** — Babel/webpack 기반 프로젝트를 SWC/Rspack으로 전환.
   "개발자 N명 × 빌드 대기시간"을 인건비로 환산해 설득한다.
   오픈소스 3개를 전환하고 전후 벤치마크를 공개하면 그것이 포트폴리오가 된다. 현금화가 가장 빠르다.
2. **유료 플러그인** — 상용 난독화, 자동 로깅/트레이싱 주입, i18n 문자열 자동 추출,
   금지 API 차단, dead code 리포터. 참고: `crates/swc_plugin*`,
   `docs/adr/00001-support-transformation-plugin-native.md`.
3. **코드 변환 SaaS** — jQuery→React, JS→TS, Vue2→Vue3, class→hooks 자동 변환.
   React 프론트 + Node(SWC) API + 결제 연동. AST 기반이라 LLM 단독보다 정확도가 높다.
4. **AI 코드 분석 엔진 / MCP** — AST로 컨텍스트를 선별해 LLM 토큰을 80~90% 절감,
   AI 생성 코드 검증 게이트, 코드 그래프 RAG. MCP 서버로 패키징하면
   Claude Code·Cursor 사용자에게 바로 판매할 수 있다. 생태계 초기라 선점 효과가 크다.
5. **교육 콘텐츠** — 유튜브(유입) → 유료 강의 → 전자책 → 멘토링 → 기업 사내교육.
6. **플레이그라운드** — `@swc/wasm-web` 기반이므로 서버 비용 없이 운영 가능.
   광고·스폰서 + Pro 기능 유료화.
7. **사내 빌드 플랫폼** — SWC 기반 분산 빌드/캐시(Nx Cloud, Turborepo 포지션). 좌석당 과금.
8. **패키지 생태계** — 유용한 무료 플러그인 배포로 평판 축적 → 후원·이직·프리랜싱 단가 상승.
9. **보안·품질 스캐너** — Rust 속도를 앞세워 "CI에서 10초 내 스캔"을 세일즈 포인트로.
10. **레거시 현대화** — SWC(구문 변환) + LLM(의미 이해)을 결합한 사내 도구로 서비스 제공.

### 권장 로드맵

```
1~2개월 : 플레이그라운드 제작 (6번) — 난이도 낮고 포트폴리오화
1~3개월 : 유튜브 시작 (5번) — 브랜딩 및 유입 확보
3개월   : 컨설팅 수주 (1번) — 실질 현금 흐름
4~6개월 : AI / MCP 엔진 (4번) — 성장 가능성 최대
6개월+  : SaaS(3번) 또는 전문 회사(10번)로 확장
```

라이선스: Apache-2.0 — 상업적 이용·수정·재배포 모두 허용되며, 저작권 고지와 NOTICE를 유지한다.

---

## 7. 참고 링크

- 포크 저장소: https://github.com/bmshin94/swc
- 업스트림: https://github.com/swc-project/swc
- 공식 문서: https://swc.rs/docs/installation/
- Babel 비교: https://swc.rs/docs/migrating-from-babel
- 벤치마크: https://swc.rs/docs/benchmark-transform
- Rust API: https://rustdoc.swc.rs/swc/
- 플러그인 ADR: `docs/adr/00001-support-transformation-plugin-native.md`
- 네이티브 애드온: `docs/native-addon-carriers.md`
- 커뮤니티 Discord: https://discord.com/invite/GnHbXTdZz6
