# Neovim 설정 스냅샷
이 설정은 Go 개발이 주목적이고 최소주의를 지향한다.

gruvbox 다크 테마와 기본 내장 기능(LSP/Treesitter)을 중심으로, 필요한 플러그인만 얇게 조합한 Neovim 환경 기록이다. 문서 내 `<leader>`는 `\\`(백슬래시) 기준이다.

> 실제 사용 흐름(예: Git 확인 순서, 복사/클립보드 패턴, 자주 쓰는 단축키)은 [`neovim-workflows.md`](./neovim-workflows.md)로 분리했다.

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
현재 버퍼에 Git hunk 표시를 붙이고 hunk 이동/preview/blame/diff 명령을 제공하는 Git 보조 계층이다. 이 설정에서는 파일 내부 변경을 줄 단위로 식별할 수 있게 해 주는 기본 관찰 도구 역할을 맡는다.

### `tpope/vim-fugitive`
`:Git` 명령 집합을 중심으로 status, blame, log, 파일 diff 같은 Git 작업을 Neovim 안에서 일관되게 수행할 수 있게 한다. 이 설정에서는 프로젝트 상태와 파일 단위 Git 정보를 조회하는 표준 인터페이스로 둔다.

### `sindrets/diffview.nvim`
프로젝트 단위 diff 뷰(working tree, 브랜치 비교, 파일 히스토리)를 패널 UI로 제공한다. 이 설정에서는 범위가 큰 변경을 파일 목록과 상세 diff로 분리해 확인할 수 있는 전용 뷰어로 배치한다.

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
