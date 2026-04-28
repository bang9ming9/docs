# Neovim 사용 흐름 메모
이 문서는 현재 [`neovim-setup.md`](./neovim-setup.md) 설정을 바탕으로, 실제 편집/검토 작업에서 반복해서 쓰는 흐름을 정리한 기록이다.

설정 자체(플러그인 역할, 로딩 전략, 전역 옵션)는 setup 문서에 두고, 여기서는 “어떤 상황에서 어떤 키와 동작을 선택하는지”에 집중한다.

## 목차

- [Git 변경을 확인할 때의 흐름](#git-변경을-확인할-때의-흐름)
- [선택/복사할 때의 흐름](#선택복사할-때의-흐름)
- [함수 시그니처를 복사할 때의 흐름](#함수-시그니처를-복사할-때의-흐름)
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
   - 파일을 빠르게 확인한 뒤에는 이전 창으로 돌아오거나 status 창으로 다시 이동해 다음 파일을 선택한다.
   - staged/worktree 기준으로 더 깊은 비교가 필요할 때만 `:Gvdiffsplit` 같은 별도 diff 명령 흐름으로 들어간다.
   - 이렇게 status를 기준점으로 두면 “이번 작업셋에서 아직 확인 안 한 파일”을 놓치지 않기 쉽다.

3. **브랜치 단위/큰 범위 비교 — `diffview.nvim`**
   - 작업 중인 변경(working tree)을 프로젝트 단위로 볼 때는 `:DiffviewOpen`을 쓴다.
   - 현재 브랜치를 `main`/`master`와 비교할 때는 `:DiffviewOpen main...HEAD`(또는 `master...HEAD`)처럼 기준점을 명시해 연다.
   - 특정 브랜치끼리는 `:DiffviewOpen base...target` 형태로 연다.
   - 여기서 핵심은 **working tree diff**(내 작업 디렉터리 변화)와 **branch diff**(두 기준점 사이 변화)를 분리해서 이해하는 것이다.
   - diffview의 파일 목록 패널에서 변경 파일을 이동하며 전체를 훑고, 변경량이 큰 파일은 상세 diff 창에서 집중해서 확인한다.

요약하면, 현재 내 흐름에서는 작은 범위는 gitsigns, 작업셋 기준 탐색은 fugitive status, 브랜치/히스토리 같은 큰 범위는 diffview로 나눠 쓴다.

## 선택/복사할 때의 흐름

Git 확인 흐름으로 변경 범위를 정리한 다음, 필요한 코드 조각을 바로 복사해 이슈/리뷰/메신저에 붙이는 작업을 자주 한다. 여기서는 플러그인 키맵이 아니라 Neovim 기본 동작 기준으로, 실제로 자주 쓰는 선택/복사 흐름만 정리한다.

### Visual mode로 영역 선택

- `v`: 문자 단위 선택. 커서를 움직이며 일반 영역을 잡을 때 쓴다.
- `V`: 줄 단위 선택. 코드 블록이나 로그처럼 “줄 단위로 통째로” 가져갈 때 쓴다.
- `<C-v>`: 블록(직사각형) 선택. 여러 줄의 같은 열만 잘라서 복사할 때 쓴다.

마우스 드래그 대신 키보드로 영역을 잡는 기본 패턴은 `v`/`V`/`<C-v>`로 진입 → 이동 키(`h`,`j`,`k`,`l`, `w`, `f` 등)로 범위 조정 → `y`(또는 `"+y`)로 복사다.

### yank와 시스템 클립보드 복사

- `y`는 delete가 아니라 **yank(복사)** 다. 기본적으로 Vim 레지스터로 들어간다.
- 시스템 클립보드로 바로 복사할 때는 `"+` 레지스터를 붙여 `"+y` 형태를 쓴다.
- Visual 선택 후 `"+y`를 누르면 선택 영역이 시스템 클립보드로 복사된다.

자주 쓰는 최소 패턴:

- 단어 복사: `"+yiw` (`iw` = 현재 단어 text object)
- 현재 줄 복사: `"+yy`
- 선택 영역 복사: `v` 또는 `V`로 선택 후 `"+y`
- 직사각형 블록 복사: `<C-v>`로 블록 선택 후 `"+y`

> 참고: `"+` 클립보드 레지스터 동작은 환경의 clipboard provider(예: macOS `pbcopy`)가 잡혀 있어야 한다. provider가 없는 환경에서는 기본 `y`로 내부 레지스터 복사만 동작할 수 있다.

## 함수 시그니처를 복사할 때의 흐름

함수 선언 전체가 아니라 필요한 조각만 빠르게 가져갈 때 아래 흐름을 반복해서 쓴다. (예: 리뷰 코멘트에 함수 이름만 붙이거나, 인자 목록만 별도 공유)

### 1) 한 줄 선언에서 `aaa()`만 복사

예시:

```go
func aaa() {
```

```solidity
function aaa() {
```

- 함수 이름 첫 글자(`a`)에 커서를 둔다.
- `yf)`를 쓰면 커서부터 `)`까지 yank한다(즉 `aaa()`까지).
- 시스템 클립보드가 필요하면 `"+yf)`를 같은 방식으로 쓴다.

여기서는 `func`/`function`/`{`를 제외하고 함수 호출 형태만 빠르게 가져오는 용도로 쓴다.

### 2) 여러 줄 시그니처에서 함수 이름부터 마지막 `)`까지 복사

예시:

```solidity
function aaa(
  uint256 a,
  uint256 b
)
```

여러 줄일 때는 한 번에 정확히 잡기보다, Visual 선택 + 괄호 매칭 이동을 조합하는 편이 실수율이 낮다.

- 함수 이름 첫 글자(`a`)에서 `v`로 선택 시작
- `f(`로 여는 괄호 `(`까지 이동한 뒤 `%`로 대응되는 닫는 괄호 `)`로 점프
- 필요한 범위가 맞는지 확인 후 `"+y`

코드 형태가 매번 같지 않아서, 이 흐름은 “함수 이름부터 닫는 괄호까지”를 눈으로 확인하면서 보수적으로 복사할 때 쓴다.

### 3) 괄호 안 인자만 / 괄호 포함 전체 복사

괄호 text object를 쓰면 인자만 따로 가져오기 쉽다.

- 괄호 안 인자만: `yi(` 또는 `yi)`
- 괄호 포함 전체: `ya(` 또는 `ya)`
- 시스템 클립보드로 바로 복사: `"+yi(`, `"+ya(`

함수 선언이 한 줄이든 여러 줄이든, 커서가 해당 괄호 쌍 내부(또는 경계) 문맥에 있으면 같은 패턴으로 재사용할 수 있다.

## 단축키 빠른 정리

### 파일 탐색기

| 키 | 동작 | 어원(mnemonic) | 모드 |
|---|---|---|---|
| `<C-n>` | Neo-tree 토글 | **n**avigation tree | Normal |

### Git — gitsigns (hunk 단위)

| 키 | 동작 | 어원(mnemonic) | 모드 |
|---|---|---|---|
| `]h` | 다음 hunk | 다음 블록으로 이동 | Normal |
| `[h` | 이전 hunk | 이전 블록으로 이동 | Normal |
| `<leader>hp` | hunk 미리보기 | **h**unk **p**review | Normal |
| `<leader>hb` | 현재 줄 full blame | **h**unk/**line** **b**lame | Normal |
| `<leader>hd` | 현재 파일 diff(index 대비) | **h**unk/file **d**iff | Normal |

### Git — fugitive (명령 통합)

| 키 | 동작 | 어원(mnemonic) | 모드 |
|---|---|---|---|
| `<leader>gs` | `:Git` status | **g**it **s**tatus | Normal |
| `<leader>gd` | `:Gvdiffsplit` | **g**it **d**iff | Normal |
| `<leader>gb` | `:Git blame` | **g**it **b**lame | Normal |
| `<leader>gl` | `:Git log --oneline` | **g**it **l**og | Normal |

### Git — diffview (프로젝트 단위)

| 키 | 동작 | 어원(mnemonic) | 모드 |
|---|---|---|---|
| `<leader>do` | `:DiffviewOpen` (working tree) | **d**iff **o**pen | Normal |
| `<leader>dc` | `:DiffviewClose` | **d**iff **c**lose | Normal |
| `<leader>dh` | `DiffviewFileHistory %` | **d**iff file **h**istory | Normal |
| `<leader>dH` | 프로젝트 전체 커밋 히스토리 | 대문자 **H**istory (전체 범위) | Normal |
| `<leader>dm` | 기본 브랜치 비교 (`:DiffviewOpen main` 또는 `master`) | **d**iff vs default branch (**m**ain/**m**aster) | Normal |
| `<leader>dB` | base/target 입력 비교 함수 (`base...target`) | **d**iff + custom **B**ranch compare | Normal |

### 마크다운

| 키 | 동작 | 어원(mnemonic) | 모드 |
|---|---|---|---|
| `<leader>mp` | `:MarkdownPreviewToggle` | **m**arkdown **p**review | Normal |

### Telescope

| 키 | 동작 | 어원(mnemonic) | 모드 |
|---|---|---|---|
| `<leader>ff` | 파일 퍼지 검색 | **f**ind **f**iles | Normal |
| `<leader>fg` | 프로젝트 전체 live grep | **f**ind by **g**rep | Normal |
| `<leader>fb` | 버퍼 목록 | **f**ind **b**uffers | Normal |
| `<leader>fh` | help tags 검색 | **f**ind **h**elp | Normal |
| `<leader>fr` | 최근 연 파일 | **f**ind **r**ecent | Normal |
| `<leader>fs` | 현재 파일 심볼 | **f**ind file **s**ymbols | Normal |
| `<leader>fS` | 워크스페이스 심볼 | **f**ind workspace **S**ymbols | Normal |
| `<leader>fd` | diagnostics 목록 | **f**ind **d**iagnostics | Normal |
| `<leader>/` | 현재 버퍼 퍼지 검색 | `/` = in-buffer search | Normal |

### LSP (`LspAttach` 시 버퍼 로컬)

| 키 | 동작 | 어원(mnemonic) | 모드 |
|---|---|---|---|
| `gd` | 정의로 점프 | **g**oto **d**efinition | Normal |
| `gD` | 선언으로 점프 | **g**oto **D**eclaration | Normal |
| `gr` | 참조 찾기 | **g**oto **r**eferences | Normal |
| `gi` | 구현으로 점프 | **g**oto **i**mplementation | Normal |
| `K` | hover 문서 | 기본 LSP hover | Normal |
| `<leader>rn` | 심볼 리네임 | **r**e**n**ame | Normal |
| `<leader>ca` | code action | **c**ode **a**ction | Normal, Visual |
| `[d` | 이전 진단 | 이전 **d**iagnostic | Normal |
| `]d` | 다음 진단 | 다음 **d**iagnostic | Normal |
| `<leader>le` | 현재 줄 진단 팝업 | **l**sp **e**rror | Normal |
| `<leader>lf` | 파일 포맷팅(async) | **l**sp **f**ormat | Normal |
