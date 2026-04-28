# Neovim 사용 흐름 메모
이 문서는 현재 [`neovim-setup.md`](./neovim-setup.md) 설정을 바탕으로, 실제 편집/검토 작업에서 반복해서 쓰는 흐름을 정리한 기록이다.

설정 자체(플러그인 역할, 로딩 전략, 전역 옵션)는 setup 문서에 두고, 여기서는 “어떤 상황에서 어떤 키와 동작을 선택하는지”에 집중한다.

## 목차

- [Git 변경을 확인할 때의 흐름](#git-변경을-확인할-때의-흐름)
- [자동완성(cmp) 사용 흐름](#자동완성cmp-사용-흐름)
- [선택/복사할 때의 흐름](#선택복사할-때의-흐름)
- [함수/시그니처 복사 흐름](#함수시그니처-복사-흐름)
- [Treesitter textobjects 사용 흐름](#treesitter-textobjects-사용-흐름)
- [단축키 빠른 정리](#단축키-빠른-정리)

## Git 변경을 확인할 때의 흐름

플러그인 이름보다 “변경 범위” 기준으로 도구를 고른다.

1. **현재 파일의 작은 변경 확인 — `gitsigns.nvim`**
   - 파일을 편집하면서 먼저 hunk 표식을 보고 변경 덩어리 위치를 빠르게 파악한다.
   - 필요하면 hunk preview/blame으로 해당 블록 맥락만 짧게 확인한다.
   - 이 단계는 “지금 파일에서 무엇을 건드렸는지”를 빠르게 점검하는 용도다.

2. **프로젝트 전체의 변경 파일 확인 — `vim-fugitive` status**
   - `<leader>gs` 또는 `:Git`으로 status 창을 연다.
   - status 창에서 변경 파일 목록을 커서로 이동하며 확인한다.
   - 파일 위에서 Enter로 열어 내용/맥락을 먼저 훑는다.
   - 비교가 필요하면 일반 Vim 방식으로 horizontal split / vertical split / tab에 열어 컨텍스트를 유지한 채 본다.
   - staged/worktree 기준으로 더 깊은 비교가 필요할 때만 `:Gvdiffsplit` 흐름으로 들어간다.

3. **브랜치 단위/큰 범위 비교 — `diffview.nvim`**
   - 작업 중인 변경(working tree)을 프로젝트 단위로 볼 때는 `:DiffviewOpen`을 쓴다.
   - 현재 로컬 설정 기준 `<leader>dm`은 `main` 비교용으로 고정해서 사용한다(`:DiffviewOpen main...HEAD`).
   - 특정 브랜치끼리는 `:DiffviewOpen base...target` 형태로 연다.
   - 핵심은 **working tree diff**와 **branch diff**를 분리해 보는 것이다.

요약하면, 작은 범위는 gitsigns, 작업셋 기준 탐색은 fugitive status, 브랜치/히스토리 같은 큰 범위는 diffview로 나눠 쓴다.

## 자동완성(cmp) 사용 흐름

Insert 모드 입력 중에는 `nvim-cmp`를 기본 인터페이스로 사용한다. 실제로는 “후보 이동 → 문서 확인 → 확정”을 짧게 반복하는 패턴이 가장 많다.

### 기본 조작 패턴

- 자동으로 뜬 목록에서 `<C-n>`/`<C-p>`로 후보 이동
- 현재 후보 확정은 `<CR>`
- 팝업이 닫혀 있으면 `<C-Space>`로 수동 호출
- 목록을 닫고 원문 입력으로 돌아갈 때는 `<C-e>`
- 후보 문서 창은 `<C-f>`/`<C-b>`로 스크롤

### 후보 출처를 읽는 방식

후보 끝 라벨을 빠르게 보고 출처를 판단한다.

- `[LSP]`: 언어 서버가 제공한 타입/메서드/심볼 후보
- `[Snippet]`: LuaSnip 기반 스니펫 후보
- `[Buffer]`: 현재(또는 열린) 버퍼 텍스트 후보
- `[Path]`: 파일 경로 후보

### 자주 쓰는 실무 패턴

1. **메서드 후보 확인**
   - `obj.` 입력 후 `[LSP]` 후보를 훑어 타입에 맞는 메서드를 확정한다.
2. **import 자동 보강**
   - 심볼 이름 일부 입력 → `[LSP]` 후보 확정 → LSP code action/organize imports 흐름으로 정리한다.
3. **snippet 확장**
   - 반복 패턴 입력 시 `[Snippet]` 후보를 먼저 확정해 구조를 만든 뒤 본문만 채운다.

### 간단한 트러블슈팅

- 팝업이 전혀 뜨지 않으면: 현재 버퍼에 LSP가 붙었는지 `:LspInfo` 먼저 확인
- LSP 후보만 비어 있으면: `:Mason`에서 해당 서버 설치 여부 확인
- 전체 상태 점검이 필요하면: `:checkhealth lsp` 확인

## 선택/복사할 때의 흐름

Git 확인 흐름으로 변경 범위를 정리한 다음, 필요한 코드 조각을 바로 복사해 이슈/리뷰/메신저에 붙이는 작업을 자주 한다. 여기서는 플러그인 키맵이 아니라 Neovim 기본 동작 기준으로, 실제로 자주 쓰는 선택/복사 흐름만 정리한다.

### Visual mode로 영역 선택

- `v`: 문자 단위 선택
- `V`: 줄 단위 선택
- `<C-v>`: 블록(직사각형) 선택

기본 패턴은 `v`/`V`/`<C-v>` 진입 → 이동 키(`h`,`j`,`k`,`l`, `w`, `f` 등)로 범위 조정 → `y`(또는 `"+y`) 복사다.

### yank와 시스템 클립보드 복사

- `y`는 Vim 레지스터 복사
- 시스템 클립보드는 `"+` 레지스터 사용 (`"+y`)

자주 쓰는 최소 패턴:

- 단어 복사: `"+yiw`
- 현재 줄 복사: `"+yy`
- 선택 영역 복사: `v` 또는 `V` 후 `"+y`
- 직사각형 복사: `<C-v>` 후 `"+y`

> 참고: `"+` 레지스터는 OS clipboard provider가 있어야 동작한다.

## 함수/시그니처 복사 흐름

기본 Vim 모션 흐름은 여전히 유효하고, 반복되는 구조 편집에서는 Treesitter textobjects 흐름이 더 빠를 때가 많다.

### 1) 기본 모션으로 빠르게 복사 (기존 패턴 유지)

- 한 줄 선언에서 `aaa()`만 복사: 함수 이름 첫 글자에서 `yf)` 또는 `"+yf)`
- 여러 줄 선언에서 이름부터 `)`까지: `v` → `f(` → `%` → `"+y`
- 괄호 안 인자만 / 괄호 포함 전체: `yi(` / `ya(` (`"+yi(`, `"+ya(`)

### 2) textobjects로 구조 단위 복사 (현재 설정에서 더 자주 사용)

- 함수 안 어디서든 함수 전체 복사: `yaf`
- 함수 본문(내부)만 복사: `yif`
- 인자 하나만 복사: `yaa` 또는 `yia`
- 인자 하나 삭제/교체 시작: `daa` 또는 `cia`

코드가 길어질수록 `f(`, `%`로 범위를 수동 조정하기보다, `af/if/aa/ia`처럼 구문 단위를 직접 선택하는 편이 재현성이 높다.

## Treesitter textobjects 사용 흐름

`nvim-treesitter-textobjects`를 켜 둔 뒤에는 함수/인자/구조체 단위 편집을 “커서 주변 구조 기준”으로 처리한다.

### 언제 쓰는가

- 함수 전체를 통째로 리뷰 코멘트에 붙일 때: `vaf` 또는 `yaf`
- 함수 본문만 가져오고 선언부는 뺄 때: `vif` 또는 `yif`
- 긴 파라미터 목록에서 한 인자만 수정/삭제할 때: `cia`, `daa`
- Go struct/interface 블록을 통째로 옮길 때: `vac` 또는 `yac`

### 선택/이동 최소 세트

- 함수: `af`(outer), `if`(inner)
- 클래스/struct/interface 블록: `ac`(outer), `ic`(inner)
- 인자: `aa`(outer), `ia`(inner)
- 이동(설정된 경우): 다음/이전 함수, 다음/이전 인자 이동 키

> 이동 키 조합은 설정 파일 기준으로 다를 수 있어, 이 문서에서는 “다음/이전 함수·인자 이동이 가능하다”는 사용 흐름만 유지한다.

## 단축키 빠른 정리

### 파일 탐색기

| 키 | 동작 | 모드 |
|---|---|---|
| `<C-n>` | Neo-tree 토글 | Normal |

### Git

| 키 | 동작 | 모드 |
|---|---|---|
| `]h` / `[h` | 다음/이전 hunk | Normal |
| `<leader>hp` | hunk 미리보기 | Normal |
| `<leader>hb` | 현재 줄 blame | Normal |
| `<leader>hd` | 현재 파일 diff(index 대비) | Normal |
| `<leader>gs` | `:Git` status | Normal |
| `<leader>gd` | `:Gvdiffsplit` | Normal |
| `<leader>gb` | `:Git blame` | Normal |
| `<leader>gl` | `:Git log --oneline` | Normal |
| `<leader>do` | `:DiffviewOpen` (working tree) | Normal |
| `<leader>dc` | `:DiffviewClose` | Normal |
| `<leader>dh` | `DiffviewFileHistory %` | Normal |
| `<leader>dH` | 프로젝트 전체 커밋 히스토리 | Normal |
| `<leader>dm` | `:DiffviewOpen main...HEAD` | Normal |
| `<leader>dB` | `base...target` 비교 입력 | Normal |

### 자동완성 (`nvim-cmp`)

| 키 | 동작 | 모드 |
|---|---|---|
| `<C-n>` / `<C-p>` | 다음/이전 후보 선택 | Insert |
| `<CR>` | 후보 확정 | Insert |
| `<C-Space>` | completion 팝업 수동 호출 | Insert |
| `<C-e>` | completion 닫기 | Insert |
| `<C-f>` / `<C-b>` | 문서 창 스크롤 | Insert |

### Treesitter textobjects

| 키 | 동작 | 모드 |
|---|---|---|
| `af` / `if` | 함수 outer/inner 선택 | Operator-pending, Visual |
| `ac` / `ic` | struct/interface/class outer/inner 선택 | Operator-pending, Visual |
| `aa` / `ia` | 인자 outer/inner 선택 | Operator-pending, Visual |
| `yaf` / `yif` | 함수 전체/본문 복사 | Normal |
| `daa` / `cia` | 인자 삭제 / 인자 교체 입력 | Normal |

### 마크다운

| 키 | 동작 | 모드 |
|---|---|---|
| `<leader>mp` | `:MarkdownPreviewToggle` | Normal |

### Telescope

| 키 | 동작 | 모드 |
|---|---|---|
| `<leader>ff` | 파일 퍼지 검색 | Normal |
| `<leader>fg` | 프로젝트 전체 live grep | Normal |
| `<leader>fb` | 버퍼 목록 | Normal |
| `<leader>fh` | help tags 검색 | Normal |
| `<leader>fr` | 최근 연 파일 | Normal |
| `<leader>fs` | 현재 파일 심볼 | Normal |
| `<leader>fS` | 워크스페이스 심볼 | Normal |
| `<leader>fd` | diagnostics 목록 | Normal |
| `<leader>/` | 현재 버퍼 퍼지 검색 | Normal |

### LSP (`LspAttach` 시 버퍼 로컬)

| 키 | 동작 | 모드 |
|---|---|---|
| `gd` / `gD` / `gr` / `gi` | 정의/선언/참조/구현 이동 | Normal |
| `K` | hover 문서 | Normal |
| `<leader>rn` | 심볼 리네임 | Normal |
| `<leader>ca` | code action | Normal, Visual |
| `[d` / `]d` | 이전/다음 진단 | Normal |
| `<leader>le` | 현재 줄 진단 팝업 | Normal |
| `<leader>lf` | 파일 포맷팅(async) | Normal |
