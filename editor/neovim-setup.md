# Neovim 설정 스냅샷
이 설정은 Go 개발이 주목적이고 최소주의를 지향한다.

gruvbox 다크 테마와 기본 내장 기능(LSP/Treesitter)을 중심으로, 필요한 플러그인만 얇게 조합한 Neovim 환경 기록이다. 문서 내 `<leader>`는 `\\`(백슬래시) 기준이다.

## 환경 정보

| 항목 | 값 |
|---|---|
| Neovim | `NVIM v0.12.1` (LuaJIT 2.1.1774896198, Release build) |
| OS | macOS (Darwin 25.3.0, Apple Silicon) |
| 플러그인 매니저 | [`folke/lazy.nvim`](https://github.com/folke/lazy.nvim) (stable 브랜치) |
| Colorscheme | [`ellisonleao/gruvbox.nvim`](https://github.com/ellisonleao/gruvbox.nvim) (`dark`) |
| Leader key | `<leader>` = `\\` |

## 목차

- [플러그인 목록](#플러그인-목록)
- [플러그인별 상세 설명](#플러그인별-상세-설명)
- [Git 확인 흐름 (현재 설정 기준)](#git-확인-흐름-현재-설정-기준)
- [선택/복사 흐름 (현재 설정 기준)](#선택복사-흐름-현재-설정-기준)
- [함수 시그니처 복사 흐름](#함수-시그니처-복사-흐름)
- [단축키 총정리](#단축키-총정리)
- [전역 설정 (`vim.opt.*`)](#전역-설정-vimopt)
- [첫 설치 가이드](#첫-설치-가이드)
- [향후 고려 중인 플러그인](#향후-고려-중인-플러그인)
- [라이선스/영감](#라이선스영감)

## 플러그인 목록

| 이름 | GitHub | 역할 | 로딩 전략 |
|---|---|---|---|
| `folke/lazy.nvim` | [링크](https://github.com/folke/lazy.nvim) | 플러그인 매니저, 지연 로딩 | 시작 시 로드 |
| `ellisonleao/gruvbox.nvim` | [링크](https://github.com/ellisonleao/gruvbox.nvim) | 컬러스킴 | 시작 시 즉시 로드 (`priority=1000`) |
| `nvim-neo-tree/neo-tree.nvim` | [링크](https://github.com/nvim-neo-tree/neo-tree.nvim) | 파일 탐색기 사이드바 | `<C-n>` 키로 로드 |
| `nvim-lua/plenary.nvim` | [링크](https://github.com/nvim-lua/plenary.nvim) | 공통 유틸 라이브러리 | 의존성 로드 |
| `nvim-tree/nvim-web-devicons` | [링크](https://github.com/nvim-tree/nvim-web-devicons) | 파일 아이콘 | 의존성 로드 |
| `MunifTanjim/nui.nvim` | [링크](https://github.com/MunifTanjim/nui.nvim) | UI 컴포넌트 | 의존성 로드 |
| `lewis6991/gitsigns.nvim` | [링크](https://github.com/lewis6991/gitsigns.nvim) | hunk 단위 변경 표시/이동 | `BufReadPre` |
| `tpope/vim-fugitive` | [링크](https://github.com/tpope/vim-fugitive) | `:Git` 기반 인터랙티브 Git | `Git`, `Gvdiffsplit`, `Glog` 명령 또는 `<leader>g*` |
| `sindrets/diffview.nvim` | [링크](https://github.com/sindrets/diffview.nvim) | 프로젝트 전체 diff/히스토리 뷰 | `Diffview*` 명령 또는 `<leader>d*` |
| `iamcco/markdown-preview.nvim` | [링크](https://github.com/iamcco/markdown-preview.nvim) | 마크다운 브라우저 프리뷰 | `ft=markdown` |
| `nvim-treesitter/nvim-treesitter` | [링크](https://github.com/nvim-treesitter/nvim-treesitter) | 트리 기반 하이라이트/들여쓰기 | `BufReadPost`, `BufNewFile` |
| `williamboman/mason.nvim` | [링크](https://github.com/williamboman/mason.nvim) | 외부 바이너리 설치/관리 | `:Mason*` 명령 |
| `williamboman/mason-lspconfig.nvim` | [링크](https://github.com/williamboman/mason-lspconfig.nvim) | Mason-LSP 연결 | mason 의존 즉시 로드 |
| `neovim/nvim-lspconfig` | [링크](https://github.com/neovim/nvim-lspconfig) | LSP 설정 및 attach 훅 | `BufReadPre`, `BufNewFile` |
| `nvim-telescope/telescope.nvim` | [링크](https://github.com/nvim-telescope/telescope.nvim) | 파일/grep/심볼/진단 퍼지 검색 | `:Telescope` 또는 `<leader>f*` |

> Mason 자동 설치 바이너리: `gopls`, `lua-language-server`

## 플러그인별 상세 설명

### `folke/lazy.nvim`
플러그인 설치/업데이트/락파일 관리를 맡는 기반 계층이다. 지연 로딩을 전제로 구성해서 시작 시간을 불필요하게 늘리지 않도록 설계했다. 이 설정에서는 키/이벤트/명령 단위로 로딩 조건을 명확히 분리해, 기능 추가 시에도 구조가 흐트러지지 않게 유지한다.

### `ellisonleao/gruvbox.nvim`
전체 UI 톤을 결정하는 테마다. `priority=1000`으로 가장 먼저 로드해 다른 플러그인의 하이라이트 기준점 역할을 하게 했다. `background=dark`를 명시해 터미널/GUI 환경에서 색감 편차를 줄인다.

### `nvim-neo-tree/neo-tree.nvim` (+ `plenary.nvim`, `nvim-web-devicons`, `nui.nvim`)
좌측 트리 중심의 파일 탐색을 담당한다. `follow_current_file`로 현재 편집 파일을 트리에서 자동 추적하고, 파일을 열면 트리를 닫아 코드 영역을 확보한다. 파일 “구조를 눈으로 훑는 작업”은 neo-tree가 맡고, “이름/내용 기반 탐색”은 telescope가 맡도록 역할을 나눴다.

### `lewis6991/gitsigns.nvim`
현재 버퍼에서 hunk 단위 변경을 빠르게 확인/이동하는 데 집중한다. 내 흐름에서는 “프로젝트 단위 비교 전에, 지금 파일에서 어떤 덩어리가 바뀌었는지”를 먼저 보는 1차 점검 도구다. gutter 표시(`│`, `_`, `‾`, `~`)와 preview/blame/diff로 작은 범위를 짧게 확인하고, 변경 범위가 커질 때 fugitive/diffview로 넘어간다.

### `tpope/vim-fugitive`
`:Git` 인터페이스를 중심으로 status, blame, log, 파일 diff 같은 “Git 조작”을 텍스트 기반으로 통합한다. 내 설정에서는 status 창을 기준점으로 두고 변경 파일을 훑는 흐름과, `:Gvdiffsplit`처럼 파일 단위 diff를 깊게 보는 흐름을 구분해 쓴다. gitsigns가 파일 내부의 작은 확인이라면, fugitive는 “이번 작업에서 무엇이 바뀌었는지”를 프로젝트 단위로 정리하는 허브에 가깝다.

### `sindrets/diffview.nvim`
브랜치 간 비교, 커밋 히스토리 탐색, 프로젝트 범위 diff를 시각적으로 다룬다. 내 사용에서는 working tree 점검(`:DiffviewOpen`)과 브랜치 간 비교(`:DiffviewOpen main`/`master`, 또는 `base...target`)를 분리해서 본다. fugitive가 현재 작업 상태 확인의 진입점이라면, diffview는 브랜치 단위 변화처럼 큰 범위를 검토하는 뷰어 역할이다.

### `iamcco/markdown-preview.nvim`
마크다운 문서를 브라우저에서 즉시 렌더링해 확인한다. Mermaid까지 포함해 문서 결과물을 빠르게 검증할 수 있다. 코드 작성과 문서 미리보기를 Neovim 내부 키맵 하나로 왕복하는 용도로 사용한다.

### `nvim-treesitter/nvim-treesitter`
정확한 구문 인식 기반 하이라이트와 들여쓰기를 제공한다. Go/Lua/웹/문서/설정 파일군을 폭넓게 `ensure_installed`로 지정해 언어별 품질 편차를 줄였다. Vim 기본 `syntax`를 fallback으로 남겨 안정성도 같이 확보했다.

### `williamboman/mason.nvim` + `williamboman/mason-lspconfig.nvim`
LSP 서버 같은 외부 실행 파일 설치를 Neovim 내부에서 관리한다. 이 설정에서는 `gopls`, `lua_ls`를 자동 보장해 새 환경에서도 부트스트랩이 단순하다. mason-lspconfig를 연결점으로 사용해 설치 상태와 LSP 활성화를 자연스럽게 이어준다.

### `neovim/nvim-lspconfig`
Neovim 0.12의 native API(`vim.lsp.config`, `vim.lsp.enable`) 기준으로 LSP를 구성했다. `gopls`에는 `staticcheck`, `gofumpt`, `unusedparams` 분석을 켜고, `lua_ls`에는 `vim` 글로벌/서드파티 검사 설정을 반영했다. 키맵은 `LspAttach`에서 버퍼 로컬로 묶어 과도한 전역 충돌을 피했다.

### `nvim-telescope/telescope.nvim`
파일 찾기, live grep, 버퍼/헬프/심볼/진단 조회를 통합한 검색 UI다. `prompt_position=top`, `sorting_strategy=ascending`, `path_display=truncate`로 결과 읽기 흐름을 단순화했다. neo-tree가 계층 탐색 도구라면 telescope는 질의 기반 탐색 도구다.

여기까지는 플러그인별 상세 설명이고, 아래부터는 실제 작업 시 자주 쓰는 흐름을 정리한다.

## Git 확인 흐름 (현재 설정 기준)

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

요약하면, 현재 내 설정에서는 작은 범위는 gitsigns, 작업셋 기준 탐색은 fugitive status, 브랜치/히스토리 같은 큰 범위는 diffview로 나눠 쓴다. 작은 변경 확인과 큰 범위 비교를 분리하면 화면 전환 부담이 줄고, split/tab을 유지한 상태에서 검토 순서를 이어가기 쉽다.

## 선택/복사 흐름 (현재 설정 기준)

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

## 함수 시그니처 복사 흐름

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


## 단축키 총정리

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

## 전역 설정 (`vim.opt.*`)

| 옵션 | 값 | 설명 |
|---|---|---|
| `number` | `true` | 줄번호 표시 |
| `wrap` | `true` | 긴 줄 자동 줄바꿈 |
| `visualbell` | `true` | 비프음 대신 시각 알림 |
| `ruler` | `true` | 커서 위치 표시 |
| `showmatch` | `true` | 괄호 짝 하이라이트 |
| `history` | `100` | 명령 히스토리 길이 |
| `fileencodings` | `utf-8, euc-kr` | 한글 인코딩 호환 |
| `backspace` | `indent,eol,start` | 삽입 모드 백스페이스 동작 |
| `title` | `true` | 터미널 타이틀 표시 |
| `shiftwidth` | `2` | 자동 들여쓰기 폭 |
| `tabstop` | `2` | 탭 표시 폭 |
| `expandtab` | `true` | Tab 입력을 공백으로 확장 |
| `autoindent` | `true` | 기본 자동 들여쓰기 |
| `smartindent` | `true` | 문맥 기반 들여쓰기 |
| `hlsearch` | `true` | 검색 결과 하이라이트 |
| `incsearch` | `true` | 입력 중 점진 검색 |
| `wildmenu` | `true` | 명령행 자동완성 메뉴 |
| `wildmode` | `longest:full,full` | Tab 자동완성 순환 방식 |
| `showmode` | `true` | 현재 모드 표시 |
| `laststatus` | `2` | 상태줄 항상 표시 |

> 참고: `syntax`는 `vim.opt` 테이블 항목이라기보다 Vim 명령(`:syntax on`) 성격에 가깝다. 이 설정에서는 Treesitter가 주 하이라이트를 담당하고, 기본 syntax는 fallback으로만 둔다.

## 첫 설치 가이드

```bash
git clone <YOUR_REPO_URL> ~/.config/nvim
nvim
```

Neovim에 진입한 뒤 다음 순서로 실행한다.

```vim
:Lazy sync
:Mason install gopls lua-language-server
```

선택 사항으로 Treesitter 파서 상태를 확인하려면:

```vim
:TSInstallInfo
```

LSP 상태 확인은 먼저 health 체크를 권장한다.

```vim
:checkhealth vim.lsp
```

필요하면 현재 버퍼에 attach된 서버를 추가로 확인한다.

```vim
:LspInfo
```

## 향후 고려 중인 플러그인

- `folke/which-key.nvim`: 리더 키 조합을 팝업으로 보여줘 키맵 발견성을 높인다.
- `nvim-lualine/lualine.nvim`: 상태줄 정보를 구조화해 현재 컨텍스트 확인을 쉽게 한다.
- `hrsh7th/nvim-cmp`: 자동완성 엔진으로 LSP/스니펫 후보를 통합한다.
- `windwp/nvim-autopairs`: 괄호/따옴표 자동 페어 입력을 보강한다.
- `kylechui/nvim-surround`: 문자열/태그 감싸기·교체·삭제 작업을 빠르게 한다.
- `numToStr/Comment.nvim`: 주석 토글을 일관된 키맵으로 제공한다.
- `ray-x/go.nvim`: Go 전용 워크플로우(테스트/빌드/도구)를 편하게 묶는다.
- `mfussenegger/nvim-dap`: 디버깅 프로토콜 기반으로 브레이크포인트/스텝 실행을 지원한다.

## 라이선스/영감

개인 설정 스냅샷 문서이며, 필요 시 자유롭게 참고해 자신의 워크플로우에 맞게 변형해서 사용하면 된다.
