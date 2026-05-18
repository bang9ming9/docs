# Neovim 설정 스냅샷

이 문서는 현재 PC에서 사용 중인 Neovim 설정을 기록한 스냅샷이다. 설정 파일은 `~/.config/nvim/init.lua`이며, 플러그인 잠금 파일은 `~/.config/nvim/lazy-lock.json`이다.

이 문서는 현재 로컬 설정에 포함된 플러그인, LSP, 자동완성, Treesitter, Mason 관리 도구를 정리한다. 실제 키 조작과 반복 작업 흐름은 [`neovim-workflows.md`](./neovim-workflows.md)에서 따로 정리한다.

문서 내 `<leader>` 표기는 현재 Neovim 기본값인 `\` 기준으로 통일한다.

## 환경 정보

| 항목 | 값 |
| --- | --- |
| Neovim | `NVIM v0.12.1` |
| OS | macOS, Apple Silicon |
| 플러그인 매니저 | `folke/lazy.nvim` |
| Colorscheme | `Mofiqul/dracula.nvim` |
| Leader key | `\` |

현재 설정에 포함된 작업 범위는 다음과 같다.

- Go 개발에 필요한 LSP, lint, formatting 흐름을 유지한다.
- Solidity/Foundry 프로젝트를 열었을 때 LSP, Treesitter, formatting 흐름이 깨지지 않게 한다.
- 문서 작성과 코드 리뷰 중 자주 쓰는 Git/diff/검색/복사 흐름을 Neovim 안에서 처리한다.
- 같은 단어를 여러 개 선택해 동시에 수정한다.

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

이 목록은 현재 `lazy-lock.json`에 기록된 플러그인 중 사용자가 직접 기능으로 설정한 항목을 정리한 것이다.

## LSP와 언어별 설정

현재 LSP 구성은 Go 중심에서 출발했지만, 실제 작업 범위에 맞춰 Solidity도 함께 다룬다.

### Go

Go 개발에서는 `gopls`를 기본 LSP로 사용한다. 정의 이동, 참조 조회, hover, code action, rename, format 같은 기본 흐름은 LSP attach 이후 버퍼 로컬 키맵으로 연결한다.

Go 설정에서는 `nvim-cmp`, `gopls`, `golangci-lint`가 각각 다른 역할을 맡는다. `nvim-cmp`는 입력 중 후보를 보여주고, `gopls`는 Go 심볼과 코드 액션을 제공한다. `golangci-lint`는 Mason으로 설치된 별도 정적 분석 도구다.

### Solidity / Foundry

Solidity 작업을 위해 Treesitter parser와 Solidity LSP를 설정한다. 현재 기준으로는 `solidity_ls_nomicfoundation`을 사용하며, `foundry.toml`, `hardhat.config.js`, `hardhat.config.ts`, `remappings.txt`, `package.json`, `.git` 같은 파일이 root 판단에 사용된다.

Solidity 파일은 저장 시 `forge fmt`가 실행되도록 구성되어 있다. 이 동작은 `BufWritePost`에서 `*.sol` 파일에만 적용된다. `forge`가 PATH에서 실행 가능하지 않으면 formatting은 실행되지 않는다.

### Lua

Neovim 설정 자체를 Lua로 작성하기 때문에 `lua_ls`를 사용한다. `vim` 글로벌을 인식하도록 설정해 Neovim 설정 파일에서 불필요한 진단이 많이 나오지 않게 한다.

## Mason으로 관리하는 외부 도구

| 도구 | 용도 | 비고 |
| --- | --- | --- |
| `gopls` | Go LSP | Go 코드 탐색, 자동완성, code action |
| `lua-language-server` | Lua LSP | Neovim 설정 작성 보조 |
| `solidity_ls_nomicfoundation` | Solidity LSP | Solidity 코드 탐색과 진단 |
| `golangci-lint` | Go lint | Go 정적 분석 |

`dlv`는 현재 Mason 설치 목록에 포함되어 있지 않고, `nvim-dap` 관련 설정도 현재 `init.lua`에 없다.

`forge`는 Mason으로 관리하지 않는다. 현재 PC에서는 `/Users/felix/.foundry/bin/forge`에 설치된 Foundry 실행 파일을 사용하며, Solidity 파일 저장 시 `forge fmt`를 실행하기 전에 `forge`가 PATH에서 실행 가능한지만 확인한다.

## 자동완성 구성

자동완성은 `nvim-cmp`를 중심으로 구성한다.

| Source | 의미 |
| --- | --- |
| LSP | 언어 서버가 제공하는 타입, 함수, 메서드, 필드 후보 |
| Buffer | 현재 또는 열린 버퍼의 텍스트 후보 |
| Path | 파일 경로 후보 |
| Cmdline | `:` 명령행 입력 후보 |
| Snippet | LuaSnip 기반 반복 입력 후보 |

현재 설정에서는 LSP와 LuaSnip 후보가 우선 그룹으로 등록되어 있고, buffer와 path 후보가 다음 그룹으로 등록되어 있다. `/`, `?` 검색 입력에는 buffer source가 연결되어 있고, `:` 명령행 입력에는 path와 cmdline source가 연결되어 있다.

## Treesitter와 textobjects

Treesitter는 하이라이트, foldexpr, indentexpr 설정에 사용된다. `nvim-treesitter-textobjects`는 함수, 인자, class/struct 단위 선택과 이동 키를 제공한다.

현재 설정된 textobject 선택 키에는 `af`, `if`, `ac`, `ic`, `aa`, `ia`가 있다. 이동 키에는 `]f`, `[f`, `]F`, `[F`, `]a`, `[a`가 있다.

`af`, `if`, `ac`, `ic`, `aa`, `ia`는 기본 Vim textobject가 아니라 현재 Treesitter textobjects 설정에서 추가한 키다.

## Git과 리뷰 도구

Git 관련 기능은 역할을 나눠서 사용한다.

| 상황 | 도구 |
| --- | --- |
| 현재 파일의 작은 변경 확인 | `gitsigns.nvim` |
| Git status, blame, 파일 단위 diff | `vim-fugitive` |
| 브랜치 범위 비교, 히스토리 diff | `diffview.nvim` |
| 파일/문자열/심볼 검색 | `telescope.nvim` |

현재 파일의 hunk 이동과 미리보기는 `gitsigns`, Git status와 blame은 `vim-fugitive`, 브랜치 범위 비교와 히스토리 확인은 `diffview.nvim` 키맵에 연결되어 있다.

## Visual Multi와 Neo-tree 키

`vim-visual-multi`는 `<C-n>`을 기본 선택 키로 사용한다.

현재 설정에서는 `<C-n>`을 visual-multi에 사용하고, Neo-tree 토글은 `<leader>e`에 매핑한다.

| 기능 | 현재 키 |
| --- | --- |
| Neo-tree 토글 | `<leader>e` |
| Visual Multi 단어 선택 | `<C-n>` |

현재 `init.lua`에서는 Neo-tree 토글이 `<leader>e`에 매핑되어 있고, `<C-n>`은 Neo-tree 토글로 매핑되어 있지 않다.

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

현재 설정을 확인할 때는 다음 항목을 확인한다.

1. `<leader>e`로 Neo-tree가 정상 토글되는지 확인한다.
2. `<C-n>`으로 visual-multi가 같은 단어를 선택하는지 확인한다.
3. visual-multi 활성화 후 `n`, `N`으로 다음/이전 매치를 추가할 수 있는지 확인한다.
4. `Esc` 또는 `q`로 visual-multi 모드를 빠져나올 수 있는지 확인한다.
5. 문서와 치트시트에 남아 있는 `Ctrl+n = Neo-tree` 설명이 없는지 확인한다.

## 현재 설정에 없는 플러그인

아래 플러그인들은 현재 `lazy-lock.json`에 없고, `init.lua`에도 설정되어 있지 않다.

| 플러그인 | 현재 상태 |
| --- | --- |
| `folke/which-key.nvim` | 설치되어 있지 않음 |
| `kylechui/nvim-surround` | 설치되어 있지 않음 |
| `numToStr/Comment.nvim` | 설치되어 있지 않음 |
| `stevearc/conform.nvim` | 설치되어 있지 않음 |
| `folke/todo-comments.nvim` | 설치되어 있지 않음 |
| `mfussenegger/nvim-dap` | 설치되어 있지 않음 |

## 설치 후 점검 흐름

현재 설정 상태를 확인할 때는 아래 명령을 사용한다.

```vim
:Lazy
:Mason
:LspInfo
:checkhealth lsp
:checkhealth nvim-treesitter
:TSUpdate
```

Go 파일에서는 `gopls` attach 상태를 확인하고, Solidity 파일에서는 Solidity LSP와 Treesitter 동작 상태를 확인한다. `.sol` 파일 저장 시에는 `forge fmt` 실행 결과를 확인한다.

## 정리

현재 Neovim 설정은 Go, Lua, Solidity, TypeScript/React, Markdown, Git diff 확인을 포함한다.

현재 문서는 `~/.config/nvim/init.lua`, `~/.config/nvim/lazy-lock.json`, Mason 설치 도구, 로컬 `forge` 설치 상태를 기준으로 작성되어 있다.
