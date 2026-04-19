# Go에서 JSON marshal한 `[]byte`가 DB에 base64처럼 저장되는 이유 (upper ORM 사례로 이해하기)

최근에 Go + upper ORM 조합으로 데이터를 저장하다가, 꽤 헷갈리는 현상을 만났다.

분명히 `json.Marshal(...)`을 한 값인데, JSON 컬럼에 넣고 나서 조회해보면 내가 기대한 JSON 객체(`{"name":"min","age":30}`)가 아니라 base64처럼 보이는 문자열이 들어가 있는 경우가 있었다. 처음에는 upper ORM의 버그처럼 보였지만, 원인을 따라가보니 핵심은 **Go의 타입 의미와 JSON 직렬화 레이어가 겹치는 방식**에 있었다.

이 글은 그 과정을 정리한 기록이다.

## 문제 상황

처음 의도는 단순하다.

- Go에서 map/struct를 `json.Marshal`해서 `[]byte`를 만든다.
- 이 값을 DB의 JSON 계열 컬럼(JSON/JSONB 등)에 저장한다.
- 조회하면 동일한 JSON 문서가 들어가 있길 기대한다.

하지만 실제로는 레이어에 따라 아래처럼 흘러갈 수 있다.

- `payload`는 “JSON 내용을 담은 바이트”이긴 하지만, Go 타입으로는 `[]byte`다.
- upper/db의 JSON wrapper나 어댑터 경로처럼, JSON 컬럼용 값으로 다시 직렬화하는 단계를 거치면 이런 현상이 발생할 수 있다.
- 이때 `encoding/json`은 `[]byte`를 raw JSON으로 넣지 않고 **base64 문자열**로 인코딩한다.
- 결과적으로 DB에는 JSON 객체가 아니라 base64 문자열 JSON이 저장될 수 있다.

즉, “이미 JSON으로 marshal된 바이트니까 그대로 저장되겠지”라는 직관이 항상 맞지 않는다.

## 처음에는 왜 upper ORM 문제처럼 보였는지

표면적으로 보면 ORM을 통과한 뒤 데이터가 기대와 다르게 바뀌었으니, ORM이 이상하게 변환한 것처럼 느껴진다.

하지만 여기서 중요한 포인트는 다음이다.

- upper ORM이 JSON 컬럼 처리 과정에서 값을 한 번 더 직렬화하는 경로를 타면(구현/설정/드라이버 조합에 따라 가능),
- 그 시점의 값 타입이 `[]byte`인 한,
- 최종 인코딩 규칙은 Go 표준 `encoding/json`의 `[]byte` 처리 규칙을 따른다.

그래서 현재 확인 가능한 범위에서는 “upper ORM이 마법처럼 고장났다”라기보다, **Go의 표준 규칙 + upper/db JSON 처리 경로(재직렬화가 개입하는 경로)가 겹쳤을 가능성이 높다**고 보는 편이 더 안전하다.

> upper ORM 내부의 모든 경로를 단정하기는 어렵지만, JSON 컬럼 저장 전에 값 재직렬화가 발생하는 경로라면 동일 현상이 재현될 가능성이 높다.

## 실제 원인 분석

### 1) Go의 `[]byte`는 “raw JSON 문서” 타입이 아니다

Go에서 `[]byte` 자체가 raw JSON으로 특별 취급되지는 않는다. 사람이 보기에 JSON 텍스트가 담겨 있어도, 그 사실만으로 “이미 완성된 JSON 문서” 타입이 되는 것은 아니다.

이 차이를 놓치면 문제가 시작된다.

- 값의 **내용(content)**: 우연히 JSON 문자열일 수 있음
- 값의 **타입(type)**: 여전히 `[]byte`

JSON 직렬화 규칙은 내용보다 타입을 우선해서 적용된다.

### 2) `encoding/json`에서 `[]byte`의 기본 동작

Go 표준 `encoding/json`은 `[]byte`를 marshal할 때 JSON 배열이나 raw object로 내보내지 않는다. 기본적으로 **base64 인코딩된 JSON 문자열**로 다룬다.

즉, 아래처럼 보이는 흐름이 가능하다.

- 1차: `map -> json.Marshal -> []byte(예: {"name":"min","age":30})`
- 2차: `[]byte`를 다시 JSON marshal
- 결과: `"eyJuYW1lIjoibWluIiwiYWdlIjozMH0="` 같은 형태(예시)

그래서 “json.Marshal된 결과를 또 JSON layer에 넣는” 순간 기대가 쉽게 깨진다.

### 3) ORM/드라이버/JSON 컬럼 경로에서 재직렬화가 겹치는 구조

실무 경로를 단순화하면 아래와 같다.

1. 애플리케이션 코드가 값을 ORM으로 전달
2. ORM(또는 어댑터/랩퍼)이 JSON 컬럼에 맞는 형태로 값 처리(필요 시 직렬화)
3. 드라이버/DB로 전달

이때 2번에서 값이 이미 `[]byte`라면, “JSON 텍스트를 그대로 전달”이 아니라 “`[]byte` 타입을 JSON 규칙대로 인코딩”하는 경로를 타기 쉽다. 그러면 base64 문자열 JSON이 만들어진다.

핵심은 재직렬화 자체가 문제가 아니라, **재직렬화 대상 타입이 `[]byte`인 상태**라는 점이다.

## 예시 코드

아래 코드는 설명을 위한 최소 예시다. 실제 upper/db API 호출부는 버전/어댑터 설정에 따라 차이가 있을 수 있으므로, 구조 중심으로 보면 된다.

### 1) 문제가 발생할 수 있는 예시

```go
package main

import (
    "encoding/json"
    "fmt"
)

func main() {
    payload, _ := json.Marshal(map[string]any{
        "name": "min",
        "age":  30,
    })

    // 이 값을 JSON 컬럼에 저장하려고 넘겼을 때
    // 내부에서 다시 JSON marshal 되면 base64 문자열처럼 들어갈 수 있는 흐름

    // 재현을 위해 2차 marshal을 직접 수행
    second, _ := json.Marshal(payload)

    fmt.Printf("payload(string): %s\n", string(payload))
    fmt.Printf("second marshal : %s\n", string(second))
    // second 예: "eyJuYW1lIjoibWluIiwiYWdlIjozMH0="
}
```

### 2) `json.RawMessage`를 사용하는 개선 예시

`json.RawMessage`는 `encoding/json` 레이어에서 “이 값은 이미 JSON으로 간주해라”는 의도를 타입으로 전달할 때 유용하다.

```go
package main

import "encoding/json"

func main() {
    payload, _ := json.Marshal(map[string]any{
        "name": "min",
        "age":  30,
    })

    raw := json.RawMessage(payload)

    // JSON 컬럼에 저장할 필드/파라미터를 raw로 전달하면,
    // encoding/json 재직렬화가 있을 때 []byte의 base64 규칙 대신
    // RawMessage의 JSON 표현을 따르게 할 수 있다.
    // (단, upper/db 경로에서는 adapter가 RawMessage를 어떻게 처리하는지 확인 필요)
    _ = raw
}
```

### 3) 애초에 struct/map 자체를 저장 대상으로 두는 예시

가능하면 가장 안전하고 의도가 분명한 방식은 **중간 `[]byte`를 만들지 않고**, JSON 문서 의미를 가진 Go 값(구조체/맵)을 그대로 넘기는 것이다.

```go
package main

import "encoding/json"

type Meta struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func main() {
    meta := Meta{Name: "min", Age: 30}

    // 혹은 map[string]any{"name":"min", "age":30}
    // ORM/드라이버가 JSON 컬럼 직렬화를 수행할 때
    // 값 의미가 "JSON 객체"로 더 명확하게 전달된다.
    _, _ = json.Marshal(meta)
}
```


짧게 덧붙이면, upper/db를 사용하는 경우에는 adapter가 제공하는 JSON/JSONB 타입과 converter를 먼저 확인하는 편이 좋다. PostgreSQL 계열에서는 JSONB 관련 타입/컨버터, MySQL 계열에서는 JSON 관련 래퍼/컨버터처럼 DB별로 제공되는 도구가 다를 수 있으니, 컬럼 타입과 adapter 문서를 맞춰서 선택하는 것이 안전하다.

## 해결 방향 정리

정리하면 아래 순서로 판단하는 것이 좋다.

1. **저장 대상의 의미를 먼저 결정**한다.
   - JSON 문서인가?
   - 바이너리 데이터인가?

2. JSON 문서라면
   - `[]byte` 대신 struct/map 기반으로 저장하거나,
   - 이미 만들어진 JSON이라면 `json.RawMessage`를 검토한다.
   - 특히 upper/db를 쓴다면 adapter가 제공하는 JSON/JSONB 타입 및 converter(JSON, JSONB, JSONMap, JSONBMap, JSONConverter, JSONBConverter 등)를 우선 검토하는 편이 더 직접적일 수 있다.

3. 바이너리 데이터라면
   - JSON 컬럼을 쓰지 말고 BLOB/BYTEA 같은 바이너리 컬럼을 사용한다.

4. ORM/adapter 경로 점검
   - JSON 컬럼 저장 전에 재직렬화가 개입하는지,
   - 해당 타입(`[]byte`, `RawMessage`, struct/map, upper wrapper/converter)이 각각 어떻게 처리되는지 테스트로 확인한다.

## 실무적으로 얻은 결론

이번 케이스의 핵심 교훈은 간단하다.

- `[]byte`는 JSON raw 본문 타입이 아니다.
- `encoding/json`에서 `[]byte`는 base64 문자열로 직렬화되는 것이 표준 동작이다.
- 따라서 “이미 `json.Marshal`한 바이트”라도 JSON 레이어를 한 번 더 통과하면 base64 문자열이 되는 것이 이상한 일이 아니다.
- upper ORM 문제처럼 보일 수 있지만, 현재 확인 가능한 범위에서는 **Go의 `[]byte` 인코딩 규칙과 upper/db JSON 처리 경로가 겹친 결과일 가능성**이 높다.

실무에서는 “JSON 문서”와 “바이너리 바이트”를 타입/스키마에서 명확히 분리해 설계하는 것이 가장 중요하다.

---

## 짧은 요약

`json.Marshal` 결과가 `[]byte`라는 사실만 믿고 JSON 컬럼에 넘기면, upper/db의 JSON wrapper/adapter 경로처럼 재직렬화가 개입하는 경우 base64 문자열 JSON으로 저장될 수 있다. 핵심은 `[]byte`가 raw JSON으로 특별 취급되지 않는다는 점이다. JSON 문서를 저장할 땐 struct/map, `json.RawMessage`, 또는 upper/db JSON/JSONB wrapper·converter를 컬럼 타입에 맞게 선택하고, 바이너리 저장 목적이면 JSON 컬럼 대신 BLOB/BYTEA를 쓰는 것이 안전하다.
