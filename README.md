# api-first-demo-spec

Todo API의 OpenAPI 스펙 레포입니다.  
이 레포의 `openapi.yaml`이 **백엔드/프론트엔드 코드의 유일한 source of truth**입니다.

---

## 이 레포의 역할

백엔드와 프론트엔드는 이 yaml에서 코드를 자동생성합니다.

```
openapi.yaml
    │
    ├── backend → ./gradlew build → Java 인터페이스 + 모델 자동생성
    └── frontend → npx orval     → TypeScript 훅 + 타입 자동생성
```

yaml이 바뀌면 양쪽에서 컴파일 에러로 즉시 감지됩니다.  
**구두 협의나 위키 문서가 아니라, 이 파일이 계약서입니다.**

---

## AI 작업 컨벤션 (CLAUDE.md)

이 레포에는 `CLAUDE.md`가 있습니다.  
Claude Code로 yaml을 수정할 때 자동으로 로드되어 아래 규칙을 강제합니다.

- operationId 네이밍 (`camelCase`, 동사+명사)
- 스키마 네이밍 (`PascalCase`, Request/Response 구분)
- 모든 프로퍼티에 `example` 필수
- `required` 배열 명시 필수
- 4XX 응답은 공통 `ErrorResponse` 스키마 사용
- Breaking change 정의 및 금지 행동

팀 컨벤션을 바꾸려면 `CLAUDE.md`를 PR로 수정합니다.

---

## CI — 자동 스펙 검증

PR 및 push 시 GitHub Actions가 `openapi.yaml`을 자동으로 검증합니다.

```
.github/workflows/validate.yml
    └── Redocly CLI lint
            ├── 잘못된 $ref 참조
            ├── operationId 누락
            ├── 4XX 응답 누락
            ├── 보안 선언 누락
            └── OpenAPI 3.0 스펙 위반
```

CI가 실패한 PR은 merge할 수 없습니다.  
로컬에서 미리 확인하려면:

```bash
npx @redocly/cli lint openapi.yaml
```

---

## 스펙 변경 가이드

### 변경 가능한 것 (Backward-compatible)

```yaml
# ✅ 필드 추가 (optional)
Todo:
  properties:
    priority:     # 새 필드 — 기존 클라이언트는 무시하면 됨
      type: integer

# ✅ 엔드포인트 추가
/todos/{id}/comments:
  get: ...
```

### 변경 시 주의가 필요한 것 (Breaking change)

```yaml
# ⚠️ 필드 이름 변경 → 기존 클라이언트 깨짐
title → name

# ⚠️ 필드 타입 변경
id: integer → id: string

# ⚠️ 엔드포인트 삭제 또는 경로 변경
```

Breaking change가 필요하면 PR 설명에 명시하고 백엔드/프론트 담당자와 사전 합의 후 진행합니다.

---

## 브랜치 전략

```
master        ← 현재 확정된 스펙. 배포된 환경과 일치.
feature/*     ← 새 기능 스펙 작업 브랜치
```

feature 브랜치는 백엔드/프론트가 submodule로 참조해서 병렬 작업할 수 있습니다.  
master merge 후 각 레포에서 submodule을 업데이트합니다.
