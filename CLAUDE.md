# OpenAPI Spec 작업 가이드

이 레포에서 `openapi.yaml`을 수정할 때 반드시 아래 규칙을 따릅니다.  
백엔드(openapi-generator)와 프론트엔드(orval)가 이 yaml에서 코드를 자동생성하기 때문에,  
컨벤션이 깨지면 양쪽 빌드가 실패합니다.

---

## 엔드포인트 필수 항목

모든 엔드포인트에 아래 항목이 있어야 합니다.

```yaml
/todos:
  post:
    operationId: createTodo      # 필수 — camelCase, 동사+명사
    summary: Create a todo       # 필수 — 한 줄 설명
    tags: [Todo]                 # 필수 — 태그 하나 이상
    responses:
      '2XX': ...                 # 필수 — 성공 응답
      '4XX': ...                 # 필수 — 에러 응답 최소 하나 (Redocly CI가 강제)
```

누락 시 CI(`Redocly lint`)가 실패하고 merge가 차단됩니다.

---

## 네이밍 규칙

### operationId

- **camelCase**, 동사로 시작
- 패턴: `{동사}{리소스}` 또는 `{동사}{리소스}{수식어}`

```yaml
# ✅
operationId: getTodos
operationId: createTodo
operationId: getTodoById
operationId: updateTodo
operationId: deleteTodo

# ❌
operationId: get_todos
operationId: GetTodos
operationId: todos_list
```

orval이 operationId 기반으로 훅 이름을 생성합니다.  
`getTodos` → `useGetTodos`, `createTodo` → `useCreateTodo`

### 스키마 이름

- **PascalCase**
- Request body: `{동사}{리소스}Request`
- Response body: 리소스 이름 그대로

```yaml
# ✅
Todo
CreateTodoRequest
UpdateTodoRequest
ErrorResponse

# ❌
createTodoRequest
todo_response
```

### 태그

- **PascalCase**, 리소스 단수형
- 같은 리소스의 엔드포인트는 같은 태그 사용

```yaml
# ✅
tags: [Todo]

# ❌
tags: [todos]
tags: [todo_api]
```

---

## 스키마 작성 규칙

### example 필수

모든 프로퍼티에 `example`을 작성합니다.  
생성된 코드의 테스트/문서에 사용됩니다.

```yaml
# ✅
title:
  type: string
  example: Buy milk

# ❌
title:
  type: string
```

### required 명시

객체 스키마는 반드시 `required` 배열을 명시합니다.  
없으면 생성된 타입에서 모든 필드가 optional이 됩니다.

```yaml
# ✅
Todo:
  type: object
  required: [id, title, completed]
  properties: ...

# ❌
Todo:
  type: object
  properties: ...
```

### 에러 응답은 ErrorResponse 스키마 사용

4XX 응답은 항상 공통 `ErrorResponse` 스키마를 참조합니다.  
직접 인라인으로 정의하지 않습니다.

```yaml
# ✅
'400':
  description: Validation error
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/ErrorResponse'

# ❌
'400':
  description: Validation error
  content:
    application/json:
      schema:
        type: object
        properties:
          error:
            type: string
```

---

## Breaking Change 규칙

### 허용 (Backward-compatible)

```yaml
# ✅ 새 필드 추가 (optional)
Todo:
  properties:
    priority:       # 기존 클라이언트는 무시함
      type: integer

# ✅ 새 엔드포인트 추가
/todos/{id}/tags:
  get: ...
```

### 금지 (Breaking change)

아래 변경은 백엔드/프론트 담당자와 **사전 합의 없이 절대 하지 않습니다.**

```yaml
# ❌ 필드 이름 변경
title → name

# ❌ 필드 타입 변경
id: integer → id: string

# ❌ required 필드 추가 (기존 클라이언트가 해당 필드 없이 요청하면 실패)
required: [id, title, completed, priority]   # priority를 required로 올리기

# ❌ 엔드포인트 경로 변경
/todos → /todo-items

# ❌ 엔드포인트 삭제

# ❌ operationId 변경 (프론트 훅 이름이 바뀜)
getTodos → listTodos
```

Breaking change가 불가피하면 PR 설명에 영향 범위를 명시하고  
백엔드/프론트 PR과 함께 올립니다.

---

## 로컬 검증

yaml 수정 후 반드시 로컬에서 먼저 확인합니다.

```bash
npx @redocly/cli lint openapi.yaml
```

CI와 동일한 규칙으로 검사합니다. 통과해야 PR을 올립니다.

---

## 작업 순서

```
1. feature 브랜치 생성
   git checkout -b feature/add-priority

2. openapi.yaml 수정

3. 로컬 lint 확인
   npx @redocly/cli lint openapi.yaml

4. PR 오픈
   - 제목: feat: add priority field to Todo
   - 설명: Breaking change 여부, 영향받는 필드/엔드포인트 명시

5. CI 통과 확인 후 merge

6. 백엔드/프론트에서 submodule 업데이트
```
