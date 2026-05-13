# Neovim 설정 스냅샷

이 문서는 현재 사용 중인 Neovim 설정을 기록한 스냅샷이다. 처음에는 Go 개발을 중심으로 필요한 플러그인만 얇게 붙이는 구성이었지만, 실제 작업 범위가 Go 백엔드, Solidity/Foundry 프로젝트, TypeScript/React 코드 확인, 문서 작성, 브랜치 리뷰까지 넓어지면서 설정도 함께 확장되었다.

따라서 이 문서는 “추천 플러그인 목록”이라기보다, 현재 로컬 설정이 어떤 문제를 해결하기 위해 구성되어 있는지 정리하는 기준 문서에 가깝다. 실제 키 조작과 반복 작업 흐름은 [`neovim-workflows.md`](./neovim-workflows.md)에서 따로 정리한다.

문서 내 `<leader>` 표기는 `\` 기준으로 통일한다.

## 환경 정보

| 항목 | 값 |
| --- | --- |
| Neovim | `NVIM v0.12.1` 기준 |
| OS | macOS, Apple Silicon |
| 플러그인 매니저 | `folke/lazy.nvim` |
| Colorscheme | `Mofiqul/dracula.nvim` |
| Leader key | `\` |

현재 설정의 핵심 방향은 단순하다.

- Go 개발에 필요한 LSP, lint, formatting 흐름을 유지한다.
- Solidity/Foundry 프로젝트를 열었을 때 LSP, Treesitter, formatting 흐름이 깨지지 않게 한다.
- 문서 작성과 코드 리뷰 중 자주 쓰는 Git/diff/검색/복사 흐름을 Neovim 안에서 처리한다.
- 새 플러그인을 많이 설치하기보다, 실제 반복 작업에서 쓰는 기능만 남긴다.

## 플러그인 구성

| 플러그인 | 역할 | 현재 사용하는 이유 |
| --- | --- | --- |
| `folke/lazy.nvim` | 플러그인 매니저 | 플러그인 설치, 업데이트, 지연 로딩을 관리한다. |
| `Mofiqul/dracula.nvim` | 컬러스킴 | 현재 UI 기준 테마로 사용한다. |
| `nvim-neo-tree/neo-tree.nvim` | 파일 탐색기 | 프로젝트 구조를 사이드바에서 확인한다. |
| `lewis6991/gitsigns.nvim` | Git hunk 표시 | 현재 파일의 변경 위치를 줄 단위로 확인한다. |
| `tpope/vim-fugitive` | Git 명령 연동 | `:Git`, blame, status, diff 확인에 사용한다. |
| `sindrets/diffview.nvim` | 브랜치/히스토리 diff | working tree와 브랜치 범위 비교를 확인한다. |
| `iamcco/markdown-preview.nvim` | Markdown preview | 문서 작성 중 렌더링 결과를 브라우저에서 확인한다. |
| `nvim-treesitter/nvim-treesitter` | 구문 기반 하이라이트 | Go, Lua, Markdown, Solidity 등에서 구문 인식을 보강한다. |
| `nvim-treesitter/nvim-treesitter-textobjects` | 구조 단위 선택/이동 | 함수, 인자, class/struct 단위 복사와 이동에 사용한다. |
| `williamboman/mason.nvim` | 외부 도구 설치 관리 | LSP, formatter, linter 같은 실행 파일을 Neovim 안에서 관리한다. |
| `williamboman/mason-lspconfig.nvim` | Mason-LSP 연결 | LSP 서버 설치와 설정 연결을 단순화한다. |
| `WhoIsSethDaniel/mason-tool-installer.nvim` | LSP 외 도구 설치 | linter, formatter 등 LSP가 아닌 도구도 같은 기준으로 관리한다. |
| `neovim/nvim-lspconfig` | LSP 설정 | Go, Lua, Solidity 등 언어 서버 연결을 담당한다. |
| `hrsh7th/nvim-cmp` | 자동완성 엔진 | Insert 모드 자동완성 UI와 후보 선택을 담당한다. |
| `hrsh7th/cmp-nvim-lsp` | LSP completion source | LSP 기반 자동완성 후보를 `nvim-cmp`에 연결한다. |
| `hrsh7th/cmp-buffer` | Buffer completion source | 현재 버퍼 기반 후보를 제공한다. |
| `hrsh7th/cmp-path` | Path completion source | 파일 경로 입력을 보조한다. |
| `hrsh7th/cmp-cmdline` | Cmdline completion source | `:` 명령행 입력을 보조한다. |
| `L3MON4D3/LuaSnip` | Snippet engine | 반복 입력 구조를 스니펫으로 보조한다. |
| `saadparwaiz1/cmp_luasnip` | Snippet completion 연결 | LuaSnip 후보를 `nvim-cmp`에서 사용할 수 있게 한다. |
| `nvim-telescope/telescope.nvim` | 검색 UI | 파일, grep, buffer, symbol, diagnostics 검색을 담당한다. |
| `mg979/vim-visual-multi` | multi-cursor 편집 | 같은 단어를 여러 개 선택해 동시에 수정할 때 사용한다. |

이 목록은 “설치하면 좋은 플러그인”을 나열한 것이 아니라, 현재 설정에서 실제 역할이 있는 플러그인만 정리한 것이다. 이후 플러그인을 추가할 때도 먼저 문서에 기능을 늘리는 것이 아니라, 반복 작업에서 불편함이 확인된 뒤 설정에 반영하는 편이 낫다.

## LSP와 언어별 설정

현재 LSP 구성은 Go 중심에서 출발했지만, 실제 작업 범위에 맞춰 Solidity도 함께 다룬다.

### Go

Go 개발에서는 `gopls`를 기본 LSP로 사용한다. 정의 이동, 참조 조회, hover, code action, rename, format 같은 기본 흐름은 LSP attach 이후 버퍼 로컬 키맵으로 연결한다.

Go 설정에서 중요한 점은 자동완성과 import 정리, lint 흐름을 서로 분리해서 보는 것이다. `nvim-cmp`는 입력 중 후보를 보여주는 역할이고, `gopls`는 Go 심볼과 코드 액션을 제공한다. `golangci-lint`는 별도의 정적 분석 도구로 보고, LSP와 같은 도구처럼 생각하지 않는 편이 혼동이 적다.

### Solidity / Foundry

Solidity 작업을 위해 Treesitter parser와 Solidity LSP를 설정한다. 현재 기준으로는 `solidity_ls_nomicfoundation`을 사용하며, Foundry 프로젝트에서는 `foundry.toml`, `remappings.txt` 같은 파일이 root 판단과 import 해석에 영향을 준다.

Solidity 파일은 저장 시 `forge fmt`가 실행되도록 구성되어 있다. 이 방식은 Foundry 프로젝트 안에서는 편리하지만, 모든 Solidity 파일에 항상 안전한 전제는 아니다. 프로젝트 루트가 잘못 잡히거나 Foundry 설정이 없는 파일을 열었을 때는 formatting 동작이 기대와 다를 수 있다. 따라서 저장 후 자동 formatting은 “현재 작업 중인 Foundry 프로젝트에 맞춘 편의 설정”으로 보는 것이 좋다.

### Lua

Neovim 설정 자체를 Lua로 작성하기 때문에 `lua_ls`를 사용한다. `vim` 글로벌을 인식하도록 설정해 Neovim 설정 파일에서 불필요한 진단이 많이 나오지 않게 한다.

## Mason으로 관리하는 외부 도구

| 도구 | 용도 | 비고 |
| --- | --- | --- |
| `gopls` | Go LSP | Go 코드 탐색, 자동완성, code action |
| `lua-language-server` | Lua LSP | Neovim 설정 작성 보조 |
| `solidity_ls_nomicfoundation` | Solidity LSP | Solidity 코드 탐색과 진단 |
| `golangci-lint` | Go lint | Go 정적 분석 |
| `forge` | Foundry formatter 실행 | 로컬 Foundry 설치 상태에 의존할 수 있음 |

`dlv`는 Go 디버깅 도구로 사용할 수 있지만, 현재 설정에서 실제로 활성화되어 있지 않다면 설치된 도구처럼 문서화하지 않는 편이 낫다. 디버깅을 자주 쓰지 않는 상태에서 `nvim-dap`과 `dlv`를 먼저 문서에 올리면, 실제 사용 흐름보다 설정 설명이 앞서갈 수 있다.

## 자동완성 구성

자동완성은 `nvim-cmp`를 중심으로 구성한다.

| Source | 의미 |
| --- | --- |
| LSP | 언어 서버가 제공하는 타입, 함수, 메서드, 필드 후보 |
| Buffer | 현재 또는 열린 버퍼의 텍스트 후보 |
| Path | 파일 경로 후보 |
| Cmdline | `:` 명령행 입력 후보 |
| Snippet | LuaSnip 기반 반복 입력 후보 |

자동완성은 편하지만, 모든 입력을 자동완성에 의존하면 오히려 흐름이 느려질 수 있다. 현재 설정에서는 LSP 후보를 우선 사용하되, 문서 작성이나 반복 코드 입력에서는 buffer/path/snippet 후보를 보조적으로 사용한다.

## Treesitter와 textobjects

Treesitter는 단순 하이라이트 도구라기보다, 코드를 구조 단위로 이해하기 위한 기반으로 사용한다. 특히 `nvim-treesitter-textobjects`를 함께 사용하면 함수, 인자, class/struct 같은 단위를 직접 선택하거나 복사할 수 있다.

예를 들어 리뷰 중 함수 전체를 메신저나 문서로 옮길 때, 수동으로 줄 범위를 맞추기보다 `yaf` 또는 `"+yaf` 같은 조작을 사용할 수 있다. 이 방식은 Go, Solidity처럼 함수 단위로 맥락을 확인하는 일이 많은 코드에서 유용하다.

다만 textobject 키는 설정에서 실제로 활성화되어 있어야 동작한다. 따라서 치트시트에 키를 적을 때는 “기본 Vim 기능”과 “현재 설정에서 추가한 textobject”를 구분해서 기록하는 편이 좋다.

## Git과 리뷰 도구

Git 관련 기능은 역할을 나눠서 사용한다.

| 상황 | 도구 |
| --- | --- |
| 현재 파일의 작은 변경 확인 | `gitsigns.nvim` |
| Git status, blame, 파일 단위 diff | `vim-fugitive` |
| 브랜치 범위 비교, 히스토리 diff | `diffview.nvim` |
| 파일/문자열/심볼 검색 | `telescope.nvim` |

이 구분을 해두면 모든 Git 작업을 하나의 플러그인으로 해결하려고 하지 않아도 된다. 파일 안의 작은 변경은 `gitsigns`, 작업셋 확인은 `fugitive`, 브랜치 단위 리뷰는 `diffview`로 나누는 편이 현재 작업 흐름에는 더 자연스럽다.

## Visual Multi와 Neo-tree 키 충돌

`vim-visual-multi`는 `<C-n>`을 기본 선택 키로 사용한다. 기존 설정에서는 `<C-n>`을 Neo-tree 토글로 사용하고 있었기 때문에 두 기능이 충돌한다.

현재 설정에서는 `<C-n>`을 visual-multi에 남기고, Neo-tree 토글을 `<leader>e`로 옮긴다. `<leader>e`는 explorer를 여는 키로 의미가 명확하고, multi-cursor 선택에서 `<C-n>`의 반복 입력 빈도가 높기 때문이다.

| 기능 | 이전 키 | 변경 후 |
| --- | --- | --- |
| Neo-tree 토글 | `<C-n>` | `<leader>e` |
| Visual Multi 단어 선택 | - | `<C-n>` |

이 선택은 `<C-n>`이 더 옳은 키라서가 아니다. `vim-visual-multi`의 기본 사용 흐름을 유지하는 편이 학습 비용이 낮고, Neo-tree는 `<leader>e`로 옮겨도 의미가 비교적 명확하다고 판단했기 때문이다.

설정 예시는 다음과 같다.

```lua
-- Neo-tree 토글 예시
vim.keymap.set("n", "<leader>e", ":Neotree toggle<CR>", {
  desc = "Toggle explorer",
})

-- vim-visual-multi 예시
{
  "mg979/vim-visual-multi",
  branch = "master",
  event = "VeryLazy",
  init = function()
    vim.g.VM_theme = "iceblue"
    vim.g.VM_highlight_matches = "underline"
    vim.g.VM_show_warnings = 1
    vim.g.VM_silent_exit = 1
  end,
}
```

적용 후에는 최소한 다음을 확인한다.

1. `<leader>e`로 Neo-tree가 정상 토글되는지 확인한다.
2. `<C-n>`으로 visual-multi가 같은 단어를 선택하는지 확인한다.
3. visual-multi 활성화 후 `n`, `N`으로 다음/이전 매치를 추가할 수 있는지 확인한다.
4. `Esc` 또는 `q`로 visual-multi 모드를 빠져나올 수 있는지 확인한다.
5. 문서와 치트시트에 남아 있는 `Ctrl+n = Neo-tree` 설명이 모두 `<leader>e`로 갱신되었는지 확인한다.

## 아직 넣지 않은 플러그인과 판단 기준

아래 플러그인들은 바로 설치할 목록이 아니라, 실제 작업 중 불편함이 반복될 때 검토할 후보로 남긴다.

| 후보 | 기대 효과 | 판단 기준 |
| --- | --- | --- |
| `folke/which-key.nvim` | leader 키 발견성 개선 | leader 조합을 자주 잊을 때 |
| `kylechui/nvim-surround` | 따옴표, 괄호, 태그 감싸기/교체 | 문자열/태그 편집이 반복될 때 |
| `numToStr/Comment.nvim` | 주석 토글 통일 | 언어별 주석 조작을 자주 할 때 |
| `stevearc/conform.nvim` | formatter 관리 통합 | `gofmt`, `forge fmt`, `prettier`, `stylua`가 흩어져 관리될 때 |
| `folke/todo-comments.nvim` | TODO/FIXME/SECURITY/AUDIT 태그 추적 | 문서, 감사, 배포 메모에서 태그 추적이 필요할 때 |
| `mfussenegger/nvim-dap` | 디버깅 UI와 breakpoint 흐름 | Go 디버깅을 실제로 자주 하게 될 때 |

현재 기준에서는 `nvim-dap`을 급하게 넣기보다 뒤로 미루는 편이 자연스럽다. 디버깅 자체가 필요 없는 것은 아니지만, 아직 `dlv`도 실제 설정에서 비활성화된 상태라면 문서와 설정이 먼저 앞서가는 구조가 된다.

반대로 `which-key`, `surround`, `Comment`, `conform`은 지금 작업 흐름과 직접 연결된다. 특히 `conform.nvim`은 Go, Solidity, TypeScript, Lua formatting을 한곳에서 관리하고 싶어질 때 검토할 만하다.

## 설치 후 점검 흐름

처음 설정을 가져온 뒤에는 아래 순서로 확인한다.

```vim
:Lazy
:Mason
:LspInfo
:checkhealth lsp
:TSInstallInfo
```

Go 파일에서는 `gopls`가 붙는지 확인하고, Solidity 파일에서는 Solidity LSP와 Treesitter parser가 정상 동작하는지 확인한다. Foundry 프로젝트에서는 `.sol` 파일 저장 시 `forge fmt`가 기대한 루트 기준으로 실행되는지도 함께 본다.

## 정리

현재 Neovim 설정은 더 이상 “Go만을 위한 최소 설정”은 아니다. Go 개발을 중심에 두되, Solidity/Foundry 프로젝트와 문서 작성, 브랜치 리뷰까지 함께 처리하는 개인 개발 환경에 가깝다.

따라서 앞으로 설정을 확장할 때는 새 플러그인을 많이 소개하는 것보다, 실제 작업 중 반복되는 불편함을 기준으로 추가 여부를 판단하는 편이 좋다. 문서도 마찬가지로 플러그인 이름보다 “왜 이 설정이 필요한지”, “어떤 작업 흐름에서 쓰는지”를 중심으로 유지하는 것이 나중에 다시 읽기 쉽다.
