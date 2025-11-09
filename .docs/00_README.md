# Grok CLI 프로젝트 문서

이 디렉토리는 Grok CLI 프로젝트의 전체 구조와 아키텍처를 이해하기 위한 문서들을 포함합니다.

## 📚 문서 목록

### 1. [프로젝트 개요](01_PROJECT_OVERVIEW.md)
- 프로젝트 목적 및 주요 기능
- 기술 스택 및 라이선스
- 주요 특징

### 2. [아키텍처](02_ARCHITECTURE.md)
- 핵심 아키텍처 패턴
- 시스템 설계
- 데이터 흐름
- 디자인 패턴

### 3. [디렉토리 구조](03_DIRECTORY_STRUCTURE.md)
- 전체 디렉토리 구조
- 각 디렉토리의 역할
- 주요 파일 설명

### 4. [핵심 컴포넌트](04_CORE_COMPONENTS.md)
- 주요 소스 파일 상세 분석
- Agent, Tools, UI 컴포넌트
- MCP 통합
- 설정 관리

### 5. [개발 가이드](05_DEVELOPMENT_GUIDE.md)
- 개발 환경 설정
- 빌드 및 실행 방법
- 테스트 및 배포
- 코딩 가이드라인

### 6. [API 통합](06_API_INTEGRATIONS.md)
- X.AI Grok API
- Morph API
- Model Context Protocol (MCP)
- Ripgrep 통합

## 🎯 빠른 시작

프로젝트를 처음 접하는 경우 다음 순서로 문서를 읽는 것을 추천합니다:

1. **프로젝트 개요** - 프로젝트가 무엇인지 이해
2. **아키텍처** - 전체적인 구조 파악
3. **디렉토리 구조** - 코드가 어떻게 조직되어 있는지 파악
4. **핵심 컴포넌트** - 주요 파일들의 역할 이해
5. **개발 가이드** - 실제 개발 시작
6. **API 통합** - 외부 서비스 통합 방법 이해

## 📊 프로젝트 통계

- **총 코드 라인 수**: ~6,231 lines (TypeScript/TSX)
- **버전**: 0.0.33
- **라이선스**: MIT
- **최소 Node.js 버전**: 18.0.0
- **권장 런타임**: Bun 1.0+

## 🔍 주요 정보 찾기

### 특정 기능 찾기
- **파일 편집 기능**: [04_CORE_COMPONENTS.md](04_CORE_COMPONENTS.md#text-editor)
- **AI Agent 로직**: [04_CORE_COMPONENTS.md](04_CORE_COMPONENTS.md#grok-agent)
- **CLI 명령어**: [01_PROJECT_OVERVIEW.md](01_PROJECT_OVERVIEW.md#cli-commands)
- **MCP 서버 관리**: [06_API_INTEGRATIONS.md](06_API_INTEGRATIONS.md#mcp)

### 개발 시작하기
- **환경 설정**: [05_DEVELOPMENT_GUIDE.md](05_DEVELOPMENT_GUIDE.md#setup)
- **빌드 방법**: [05_DEVELOPMENT_GUIDE.md](05_DEVELOPMENT_GUIDE.md#build)
- **디버깅**: [05_DEVELOPMENT_GUIDE.md](05_DEVELOPMENT_GUIDE.md#debugging)

## 🤝 기여하기

코드를 수정하기 전에 다음 문서를 반드시 읽어주세요:
- [아키텍처](02_ARCHITECTURE.md) - 설계 원칙 이해
- [개발 가이드](05_DEVELOPMENT_GUIDE.md) - 코딩 표준 및 베스트 프랙티스

## 📝 문서 업데이트

이 문서들은 프로젝트 구조나 주요 기능이 변경될 때마다 함께 업데이트되어야 합니다.

**마지막 업데이트**: 2025-11-09
**분석 기준 버전**: 0.0.33
**Git Commit**: 12a799f
