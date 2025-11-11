# Grok CLI - 완전한 문서 인덱스

## 문서 목록

### 기본 문서

1. **README_ko.md** - 로컬 모델 커스터마이징 가이드
   - 프로젝트 구조 분석
   - API 호출 메커니즘
   - 로컬 모델 통합 방법
   - 설정 관리 시스템

### 핵심 아키텍처 문서

2. **01-architecture-overview.md** - 아키텍처 개요
   - 고수준 아키텍처
   - 핵심 레이어 (Presentation, Application, Domain, Infrastructure)
   - 데이터 흐름
   - 컴포넌트 간 통신
   - 확장성 고려사항

3. **02-api-client-analysis.md** - API 클라이언트 상세 분석
   - GrokClient 클래스
   - OpenAI SDK 통합
   - 메시지 형식 (System, User, Assistant, Tool)
   - 도구 호출 (Function Calling)
   - 스트리밍 구현
   - 웹 검색 통합
   - 에러 처리
   - 로컬 모델 호환성

### 핵심 시스템 분석

4. **03-agent-system.md** - 에이전트 시스템 상세 분석
   - GrokAgent 클래스 구조
   - 에이전트 루프 메커니즘
   - 메시지 처리 방식 (블로킹/스트리밍)
   - 도구 실행 오케스트레이션
   - 토큰 카운팅
   - 중단 메커니즘
   - 채팅 히스토리 관리
   - MCP 통합

5. **04-tools-system.md** - 도구 시스템 상세 분석
   - TextEditorTool (view, create, strReplace)
   - BashTool (명령 실행)
   - SearchTool (텍스트/파일 검색)
   - TodoTool (TODO 리스트)
   - MorphEditorTool (고속 편집)
   - ConfirmationTool (사용자 확인)
   - 도구 관리 시스템
   - 도구 추가 가이드

6. **05-settings-management.md** - 설정 관리 상세 분석
   - SettingsManager 클래스
   - 설정 계층 구조 (환경변수 > CLI > 프로젝트 > 사용자 > 기본값)
   - 사용자 설정 (User Settings)
   - 프로젝트 설정 (Project Settings)
   - 우선순위 처리
   - 모델 관리
   - 보안 및 검증

### UI 및 사용자 경험

7. **06-ui-structure.md** - UI/UX 구조 상세 분석
   - Ink/React 아키텍처
   - 핵심 컴포넌트 (ChatInterface, ChatHistory, ChatInput, LoadingSpinner, etc.)
   - 입력 처리 훅 (useInputHandler, useInputHistory, useEnhancedInput)
   - 스트리밍 UI 업데이트
   - 확인 대화상자
   - 스타일링 및 테마
   - 성능 최적화

### 외부 통합

8. **07-mcp-integration.md** - MCP 통합 상세 분석
   - MCP 프로토콜 이해
   - 전송 타입 (Stdio, HTTP, SSE)
   - MCPManager 클래스
   - 서버 관리
   - 도구 통합
   - 설정 및 초기화
   - 실제 예제 (Linear, GitHub, 커스텀)
   - 보안 및 에러 처리

### 전체 흐름

9. **08-code-flow.md** - 완전한 코드 흐름 분석
   - 프로그램 시작 흐름
   - 대화형 모드 (Interactive Mode)
   - 헤드리스 모드 (Headless Mode)
   - 메시지 처리 흐름
   - 파일 작업 흐름 (확인 시스템)
   - 스트리밍 응답 흐름
   - 에러 처리 흐름
   - MCP 도구 실행 흐름
   - 전체 통신 다이어그램

## 빠른 참고

### 특정 기능 찾기

| 기능 | 문서 |
|------|------|
| API 키 관리 | 05-settings-management.md |
| 파일 편집 | 04-tools-system.md > TextEditorTool |
| Bash 명령 실행 | 04-tools-system.md > BashTool |
| 파일 검색 | 04-tools-system.md > SearchTool |
| UI 컴포넌트 | 06-ui-structure.md |
| 에이전트 루프 | 03-agent-system.md > 에이전트 루프 메커니즘 |
| 도구 호출 | 03-agent-system.md > 도구 실행 오케스트레이션 |
| MCP 서버 추가 | 07-mcp-integration.md > 실제 예제 |
| 모델 변경 | 05-settings-management.md > 모델 관리 |
| 로컬 모델 사용 | README_ko.md, 02-api-client-analysis.md > 로컬 모델 호환성 |

### 학습 순서

**초보자용:**
1. 01-architecture-overview.md - 전체 구조 이해
2. README_ko.md - 기본 설정 및 커스터마이징
3. 08-code-flow.md - 실제 흐름 이해

**중급자용:**
4. 02-api-client-analysis.md - API 통신 이해
5. 03-agent-system.md - 에이전트 동작 원리
6. 04-tools-system.md - 도구 구현 이해

**고급자용:**
7. 05-settings-management.md - 설정 시스템
8. 06-ui-structure.md - UI 커스터마이징
9. 07-mcp-integration.md - MCP 서버 통합

## 문서 통계

| 항목 | 수량 |
|------|------|
| 총 문서 수 | 9개 |
| 총 라인 수 | 7,093줄 |
| 평균 문서 크기 | 788줄 |
| 코드 예제 | 200+ 개 |
| 다이어그램 | 30+ 개 |

## 주요 특징

각 문서는 다음을 포함합니다:

✓ 상세한 설명 및 개요
✓ 클래스/함수 구조 정의
✓ 실제 코드 예제
✓ 흐름도 및 다이어그램
✓ 타입 정의
✓ 사용 예제
✓ 모범 사례 및 팁
✓ 에러 처리 방법

## 기여 및 유지보수

문서 업데이트 시 다음을 확인하세요:

1. 코드와 일치하는지 확인
2. 예제가 실행 가능한지 확인
3. 링크가 올바른지 확인
4. 문서 간 교차 참조 유지
5. 목차 업데이트

## 관련 리소스

- **소스 코드**: `/src/` 디렉토리
- **설정 파일**: `~/.grok/user-settings.json`, `.grok/settings.json`
- **타입 정의**: `src/types/index.ts`
- **상수 정의**: `src/grok/tools.ts`, `src/mcp/config.ts`

---

**마지막 업데이트**: 2025-11-11
**한국어 버전**: ✓
**영문 버전**: 계획 중
