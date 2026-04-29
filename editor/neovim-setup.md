# Neovim 설정 스냅샷

이 설정은 Go 개발을 중심으로, 필요한 플러그인만 얇게 조합해 유지하는
현재 Neovim 구성 기록이다.

gruvbox 다크 테마와 기본 내장 기능(LSP/Treesitter)을 축으로 두고,
문서 내 키 표기는 `<leader>` = `\\`(백슬래시) 기준으로 통일한다.

> 실제 사용 순서와 조작 메모는
> [`neovim-workflows.md`](./neovim-workflows.md)에서 따로 관리한다.

## 환경 정보

| 항목 | 값 |
| --- | --- |
| Neovim | `NVIM v0.12.1` (LuaJIT 2.1.1774896198, Release build) |
| OS | macOS (Darwin 25.3.0, Apple Silicon) |
| 플러그인 매니저 | [`folke/lazy.nvim`](https://github.com/folke/lazy.nvim) (stable 브랜치) |
| Colorscheme | [`ellisonleao/gruvbox.nvim`](https://github.com/ellisonleao/gruvbox.nvim) (`dark`) |
| Leader key | `<leader>` = `\\` |

## 목차

- [플러그인 목록](#플러그인-목록)
- [플러그인별 상세 설명](#플러그인별-상세-설명)
- [Mason 관리 외부 바이너리](#mason-관리-외부-바이너리)
- [플러그인 구성/안전성 메모](#플러그인-구성안전성-메모)
- [전역 설정 (`vim.opt.*`)](#전역-설정-vimopt)
- [첫 설치 가이드](#첫-설치-가이드)
- [향후 고려 중인 플러그인](#향후-고려-중인-플러그인)
- [라이선스/영감](#라이선스영감)

## 플러그인 목록

| 이름 | GitHub | 역할 | 로딩 전략 |
| --- | --- | --- | --- |
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
| `nvim-treesitter/nvim-treesitter-textobjects` | [링크](https://github.com/nvim-treesitter/nvim-treesitter-textobjects) | 함수/인자/클래스 단위 textobject 선택·이동 | treesitter 의존 로드 |
| `williamboman/mason.nvim` | [링크](https://github.com/williamboman/mason.nvim) | 외부 바이너리 설치/관리 | `:Mason*` 명령 |
| `williamboman/mason-lspconfig.nvim` | [링크](https://github.com/williamboman/mason-lspconfig.nvim) | Mason-LSP 연결 | mason 의존 즉시 로드 |
| `WhoIsSethDaniel/mason-tool-installer.nvim` | [링크](https://github.com/WhoIsSethDaniel/mason-tool-installer.nvim) | LSP 외 도구(lint/debug 등) 자동 설치 | mason 의존 즉시 로드 |
| `neovim/nvim-lspconfig` | [링크](https://github.com/neovim/nvim-lspconfig) | LSP 설정 및 attach 훅 | `BufReadPre`, `BufNewFile` |
| `hrsh7th/nvim-cmp` | [링크](https://github.com/hrsh7th/nvim-cmp) | Insert 모드 자동완성 엔진 | `InsertEnter`, `CmdlineEnter` |
| `hrsh7th/cmp-nvim-lsp` | [링크](https://github.com/hrsh7th/cmp-nvim-lsp) | LSP completion source | cmp 의존 로드 |
| `hrsh7th/cmp-buffer` | [링크](https://github.com/hrsh7th/cmp-buffer) | 현재/열린 버퍼 completion source | cmp 의존 로드 |
| `hrsh7th/cmp-path` | [링크](https://github.com/hrsh7th/cmp-path) | 파일 경로 completion source | cmp 의존 로드 |
| `hrsh7th/cmp-cmdline` | [링크](https://github.com/hrsh7th/cmp-cmdline) | `:` 명령행 completion source | cmp 의존 로드 |
| `L3MON4D3/LuaSnip` | [링크](https://github.com/L3MON4D3/LuaSnip) | 스니펫 엔진 | cmp/snippet 사용 시 로드 |
| `saadparwaiz1/cmp_luasnip` | [링크](https://github.com/saadparwaiz1/cmp_luasnip) | LuaSnip completion source 연결 | cmp 의존 로드 |
| `nvim-telescope/telescope.nvim` | [링크](https://github.com/nvim-telescope/telescope.nvim) | 파일/grep/심볼/진단 퍼지 검색 | `:Telescope` 또는 `<leader>f*` |

## 플러그인별 상세 설명

### `folke/lazy.nvim`

플러그인 설치/업데이트/락파일 관리를 맡는 기반 계층이다.
이 설정은 키/이벤트/명령 조건으로 로딩 시점을 분리해,
시작 비용과 기능 확장 시 복잡도를 함께 관리한다.

### `ellisonleao/gruvbox.nvim`

전체 UI 기준 테마다.
`priority=1000`으로 먼저 로드해 하이라이트 기준을 고정하고,
`background=dark`로 터미널/GUI 간 색감 편차를 줄인다.

### `nvim-neo-tree/neo-tree.nvim` (+ `plenary.nvim`, `nvim-web-devicons`, `nui.nvim`)

사이드바 파일 트리를 제공한다.
`follow_current_file`를 켜서 현재 버퍼와 트리 위치를 맞추고,
파일 오픈 시 트리를 닫는 옵션으로 편집 영역을 확보한다.

### `lewis6991/gitsigns.nvim`

버퍼 내 Git hunk 표시와 hunk 단위 preview/blame/diff 명령을 제공한다.
이 구성에서는 현재 열어 둔 파일 안의 변경을 촘촘히 보는 역할에 가깝다.

### `tpope/vim-fugitive`

`:Git` 중심의 status/blame/log/diff 인터페이스를 제공한다.
이 문서 기준으로는 작업셋(status) 확인과 Git 명령 진입점 역할을 맡는다.

### `sindrets/diffview.nvim`

프로젝트 단위 diff 및 히스토리 뷰를 패널 UI로 제공한다.
브랜치 비교/프로젝트 전체 변경 검토처럼 범위가 큰 diff를 다룰 때 중심이 된다.

세 플러그인 모두 diff/blame 계열 기능이 일부 겹쳐 보일 수 있지만,
실제 사용 범위를 나누면 다음처럼 책임이 분리된다.

- `gitsigns`: 버퍼 hunk
- `fugitive`: Git 작업셋/명령 진입
- `diffview`: 프로젝트/브랜치 비교

다만 실제 워크플로우에서 거의 쓰지 않는 플러그인이 생기면
제거 후보로 두고 정리하는 편이 유지보수에 유리하다.

### `iamcco/markdown-preview.nvim`

마크다운 렌더링 결과를 브라우저에서 확인한다.
문서 편집 결과(예: Mermaid 포함)를 Neovim 키맵에서 바로 점검하는 용도로 둔다.

운영 관점에서는 일반 Lua 플러그인보다 신뢰 경계가 넓다.
(Node/Yarn 기반 설치/빌드, preview 서버 실행)
따라서 위험하다고 단정하기보다,
preview를 외부 네트워크에 열지 않고 localhost 중심으로 쓰는 기준을 둔다.
예를 들면 `mkdp_open_to_the_world = 0`,
`mkdp_open_ip = "127.0.0.1"`, `mkdp_auto_start = 0` 같은 설정 의도를 남기고,
자동 실행보다 필요 시 수동 실행을 기본으로 삼는 편이 관리하기 쉽다.

### `nvim-treesitter/nvim-treesitter` + `nvim-treesitter/nvim-treesitter-textobjects`

둘은 중복이 아니라 확장 관계다.
`nvim-treesitter`는 구문 트리 기반 하이라이트/들여쓰기의 기반 계층이고,
`nvim-treesitter-textobjects`는 그 트리를 재사용해
함수·인자·클래스 단위 선택/이동/조작을 추가한다.

또한 Treesitter는 parser 설치/업데이트가 함께 따라온다.
플러그인 업데이트와 parser 상태를 같이 점검해야
버전 불일치로 인한 동작 차이를 줄일 수 있다.
로딩 전략(lazy-loading 포함)도 단정적으로 보기보다,
실제 설정에서 로딩 시점과 업데이트 동작을 함께 확인하는 편이 안전하다.

### `williamboman/mason.nvim` + `williamboman/mason-lspconfig.nvim` + `WhoIsSethDaniel/mason-tool-installer.nvim`

이 조합은 중복이라기보다 계층 분리다.

- `mason.nvim`: 외부 실행 파일 설치/관리 기반
- `mason-lspconfig.nvim`: Mason 설치 LSP와 `nvim-lspconfig` 연결
- `mason-tool-installer.nvim`: LSP 외 도구 보장 설치

Mason 계층은 외부 바이너리를 내려받아 로컬 실행 환경에 연결하므로,
운영 관점에서는 필요한 도구만 설치 목록에 유지하는 기준이 중요하다.

### `neovim/nvim-lspconfig`

Neovim 0.12의 native API(`vim.lsp.config`, `vim.lsp.enable`) 기준으로 LSP를 구성했다.
`gopls`의 `staticcheck`, `gofumpt`, `unusedparams` 분석을 켜고,
`lua_ls`는 `vim` 글로벌/서드파티 검사 설정을 반영했다.
키맵은 `LspAttach`에서 버퍼 로컬로 등록한다.

### `hrsh7th/nvim-cmp` + completion source(`cmp-*`) + `LuaSnip`

`nvim-cmp`가 자동완성 UI/선택/확정을 담당하고 source 플러그인이 후보를 공급한다.
Insert 모드/LSP 기반 자동완성의 중심은 `nvim-cmp`다.

`cmp-cmdline`은 `:` 명령행 completion까지 cmp UI로 확장하는 역할이고,
`wildmenu`/`wildmode`는 Vim 기본 명령행 completion 동작이다.
둘을 함께 둘 수는 있지만 입력 흐름이 어색하거나 UI가 중복된다고 느껴지면,
`cmp-cmdline` 또는 `wildmenu`/`wildmode` 중 하나를 줄여 정리할 수 있다.

### `nvim-telescope/telescope.nvim`

파일/grep/버퍼/심볼/진단 조회를 통합한 질의형 검색 UI다.
`prompt_position=top`, `sorting_strategy=ascending`, `path_display=truncate`를
기본값으로 사용한다.

`neo-tree`와 겹쳐 보일 수 있지만 탐색 방식이 다르다.
`neo-tree`는 디렉터리 구조를 눈으로 훑는 사이드바 탐색기이고,
`telescope`는 파일명/본문/심볼/진단을 질의로 찾는 picker라서
상호 보완적으로 사용할 수 있다.

## Mason 관리 외부 바이너리

아래는 현재 설정에서 Mason 계층으로 관리하는 바이너리 목록이다.

| 도구 | 용도 | 관리 계층 |
| --- | --- | --- |
| `gopls` | Go LSP 서버 | `mason-lspconfig.nvim` |
| `lua-language-server` (`lua_ls`) | Lua LSP 서버 | `mason-lspconfig.nvim` |
| `golangci-lint` | Go 린터 (현재 워크플로우에서 직접 사용 가능) | `mason-tool-installer.nvim` |
| `dlv` | Go 디버거(Delve): CLI 직접 사용 또는 향후 DAP 연동 후보 | `mason-tool-installer.nvim` |

## 플러그인 구성/안전성 메모

이 문서의 플러그인들은 Neovim 생태계에서 널리 사용되는 편이지만,
플러그인은 결국 로컬에서 실행되는 코드이므로 안전성을 무조건 단정하지 않는다.
운영/관리 기준은 아래처럼 둔다.

- 업데이트는 `lazy-lock.json`을 유지한 상태에서 변경 사항을 확인하며 단계적으로 진행한다.
- Mason 계층처럼 외부 바이너리를 내려받는 구성은 설치 목록을 최소화하고, 실제 워크플로우에 연결된 도구만 유지한다.
- `markdown-preview.nvim`처럼 Node/Yarn 기반 빌드나 preview 서버 동작이 있는 경우 자동 실행/네트워크 공개 범위를 보수적으로 관리한다.
- Treesitter parser처럼 별도 설치/업데이트되는 구성요소는 플러그인 업데이트와 함께 점검한다.
- 기능이 겹쳐 보이는 플러그인은 중복 설치로 단정하기보다 역할 분리 여부를 먼저 확인하고, 장기간 미사용이면 정리 후보로 둔다.

## 전역 설정 (`vim.opt.*`)

| 옵션 | 값 | 설명 |
| --- | --- | --- |
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

> 참고: `syntax`는 `vim.opt` 항목이라기보다 Vim 명령(`:syntax on`) 성격에 가깝다.
> 이 설정에서는 Treesitter를 기본 하이라이트로 두고, Vim syntax는 fallback으로만 둔다.

## 첫 설치 가이드

```bash
git clone <YOUR_REPO_URL> ~/.config/nvim
nvim
```

초기 부트스트랩은 아래 순서로 진행된다.

1. 첫 실행 시 `lazy.nvim`이 lock 기준으로 플러그인을 설치/동기화한다.
2. Go/Lua 파일을 열면 `mason-lspconfig` 경로에서 필요한 서버(`gopls`, `lua_ls`) 보장 설치를 처리한다.
3. LSP 외 도구(예: `golangci-lint`, `dlv`)는 `mason-tool-installer` 보장 목록으로 관리한다.
   `golangci-lint`는 즉시 활용 가능하고,
   `dlv`는 현재 구성에서 주로 CLI 용도이며
   Neovim 내부 디버깅 연동은 `nvim-dap` 도입 시점에 확장한다.

즉, 이 설정은 별도 수동 설치를 최소화하고
파일을 열어 작업을 시작하는 흐름에서 필요한 도구를 채우는 방식을 전제로 한다.

설치/연결 상태 확인은 아래 순서로 점검한다.

```vim
:Mason
:LspInfo
:checkhealth lsp
```

선택 사항으로 Treesitter 파서 상태를 확인하려면:

```vim
:TSInstallInfo
```

## 향후 고려 중인 플러그인

- `folke/which-key.nvim`: 리더 키 조합을 팝업으로 보여줘 키맵 발견성을 높인다.
- `nvim-lualine/lualine.nvim`: 상태줄 정보를 구조화해 현재 컨텍스트 확인을 쉽게 한다.
- `windwp/nvim-autopairs`: 괄호/따옴표 자동 페어 입력을 보강한다.
- `kylechui/nvim-surround`: 문자열/태그 감싸기·교체·삭제 작업을 빠르게 한다.
- `numToStr/Comment.nvim`: 주석 토글을 일관된 키맵으로 제공한다.
- `ray-x/go.nvim`: Go 전용 워크플로우(테스트/빌드/도구)를 편하게 묶는다.
- `mfussenegger/nvim-dap`: 디버깅 프로토콜 기반으로 브레이크포인트/스텝 실행을 지원한다.

## 라이선스/영감

개인 설정 스냅샷 문서이며,
필요 시 자유롭게 참고해 자신의 워크플로우에 맞게 변형해서 사용하면 된다.
