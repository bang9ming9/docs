# Neovim 사용 흐름 메모
이 문서는 [`neovim-setup.md`](./neovim-setup.md) 구성 위에서, 실제 작업 때 반복해서 쓰는 조작 흐름만 정리한 기록이다.

설정값과 플러그인 구성은 setup 문서에서 확인하고, 여기서는 “언제 어떤 흐름을 고르는지”를 중심으로 본다.

## 목차

- [Git 변경을 확인할 때의 흐름](#git-변경을-확인할-때의-흐름)
- [자동완성(cmp) 사용 흐름](#자동완성cmp-사용-흐름)
- [선택/복사할 때의 흐름](#선택복사할-때의-흐름)
- [함수/시그니처 복사 흐름](#함수시그니처-복사-흐름)
- [Treesitter textobjects 사용 흐름](#treesitter-textobjects-사용-흐름)
- [단축키 빠른 정리](#단축키-빠른-정리)

## Git 변경을 확인할 때의 흐름

도구 이름보다 변경 범위 기준으로 선택한다.

1. **현재 파일의 작은 변경 확인 — `gitsigns.nvim`**
   - 편집 중 hunk 표식으로 변경 위치를 먼저 훑는다.
   - 필요할 때만 hunk preview/blame으로 해당 블록 맥락을 짧게 본다.

2. **프로젝트 작업셋 확인 — `vim-fugitive` status**
   - `<leader>gs` 또는 `:Git`으로 status를 열고 변경 파일 목록을 확인한다.
   - 파일을 열어 맥락을 읽고, 필요하면 split/tab으로 비교를 이어 간다.
   - staged/worktree를 더 깊게 비교할 때 `:Gvdiffsplit`을 사용한다.

3. **브랜치/히스토리 범위 비교 — `diffview.nvim`**
   - working tree 전체 확인은 `:DiffviewOpen`을 사용한다.
   - 현재 문서 기준 키맵에서 `<leader>dm`은 `:DiffviewOpen main...HEAD`에 고정돼 있다.
   - 특정 브랜치 비교는 `:DiffviewOpen base...target` 형태로 연다.

요약하면, 파일 내부는 gitsigns, 작업셋 확인은 fugitive status, 브랜치/히스토리는 diffview로 분리한다.

## 자동완성(cmp) 사용 흐름

Insert 모드에서는 `nvim-cmp`를 기본 인터페이스로 쓴다. 가장 자주 반복하는 순서는 **후보 이동 → 필요 시 문서 확인 → 확정**이다.

### 기본 조작 패턴

- `<C-n>` / `<C-p>`: 후보 이동
- `<CR>`: 현재 후보 확정
- `<C-Space>`: 팝업 수동 호출
- `<C-e>`: 팝업 닫고 원문 입력으로 복귀
- `<C-f>` / `<C-b>`: 후보 문서 스크롤
- 현재 설정 기준으로 `<Tab>` / `<S-Tab>`은 cmp 후보 이동 키로 별도 매핑하지 않는다.

### 후보 출처를 읽는 방식

- `[LSP]`: 언어 서버 후보(타입/메서드/심볼)
- `[Snippet]`: LuaSnip 기반 스니펫 후보
- `[Buffer]`: 현재(또는 열린) 버퍼 텍스트 후보
- `[Path]`: 파일 경로 후보

### 자주 쓰는 패턴 메모

1. **메서드/필드 탐색**
   - `obj.` 입력 뒤 `[LSP]` 후보를 이동하며 바로 확정한다.
2. **import 정리로 이어지는 입력**
   - 심볼 일부 입력 → `[LSP]` 후보 확정 → 필요 시 code action/organize imports로 마무리한다.
3. **반복 구조 입력**
   - `[Snippet]` 후보로 틀을 먼저 만든 뒤 본문만 채운다.

### 짧은 점검 순서

- 후보가 전혀 안 뜨면 `:LspInfo`
- LSP 후보만 비면 `:Mason`
- 전체 상태 점검은 `:checkhealth lsp`

## 선택/복사할 때의 흐름

리뷰/이슈/메신저로 코드 조각을 옮길 때는 기본 Vim 선택 동작을 우선 사용한다. 여기서는 즉석 복사에 자주 쓰는 최소 패턴만 남긴다.

### Visual mode로 영역 선택

- `v`: 문자 단위
- `V`: 줄 단위
- `<C-v>`: 블록(직사각형)

기본 패턴은 `v`/`V`/`<C-v>` 진입 → 이동 키(`h`,`j`,`k`,`l`, `w`, `f` 등)로 범위 조정 → `y`(또는 `"+y`)다.

### yank와 시스템 클립보드 복사

- `y`: Vim 레지스터 복사
- `"+`: 시스템 클립보드 레지스터

자주 쓰는 최소 패턴:

- 단어 복사: `"+yiw`
- 현재 줄 복사: `"+yy`
- 선택 영역 복사: `v` 또는 `V` 후 `"+y`
- 직사각형 복사: `<C-v>` 후 `"+y`

> 참고: `"+` 레지스터는 OS clipboard provider가 있어야 동작한다.

## 함수/시그니처 복사 흐름

기본 Vim 모션은 fallback으로 계속 쓰고, 반복 구조 편집/복사는 Treesitter textobjects를 우선 사용한다.

### 1) 기본 모션으로 즉석 복사 (fallback)

- 한 줄 선언에서 `aaa()`만 복사: 함수 이름 첫 글자에서 `yf)` 또는 `"+yf)`
- 여러 줄 선언에서 이름부터 `)`까지: `v` → `f(` → `%` → `"+y`
- 괄호 안 인자만 / 괄호 포함 전체: `yi(` / `ya(` (`"+yi(`, `"+ya(`)

### 2) textobjects로 구조 단위 복사/편집 (반복 작업)

- 함수 안 어디서든 함수 전체 복사: `yaf`
- 함수 본문(내부)만 복사: `yif`
- 인자 하나만 선택/복사: `yaa`, `yia`
- 인자 하나 삭제/교체 시작: `daa`, `cia`

> `yaf`는 Ex 명령(`:yaf`)이 아니라 Normal mode 키 입력이다. 시스템 클립보드로 바로 보낼 때는 `"+yaf`를 사용한다. 또한 `af` textobject 자체는 현재 설정에 매핑/활성화돼 있어야 동작한다.

## Treesitter textobjects 사용 흐름

`nvim-treesitter-textobjects`는 함수/인자/구조체 같은 단위를 직접 잡아서 편집할 때 사용한다. 긴 코드에서 범위를 수동으로 맞추는 시간을 줄이는 데 목적이 있다.

### 언제 쓰는가

- 함수 전체를 리뷰 코멘트로 보낼 때: `vaf`, `yaf`
- 선언부를 빼고 함수 본문만 다룰 때: `vif`, `yif`
- 긴 파라미터 목록에서 인자 하나만 수정할 때: `cia`, `daa`
- Go struct/interface 블록 단위로 옮길 때: `vac`, `yac`

### 자주 쓰는 단위

- 함수: `af`(outer), `if`(inner)
- 클래스/struct/interface: `ac`(outer), `ic`(inner)
- 인자: `aa`(outer), `ia`(inner)

> 이동 키 조합은 설정 파일마다 달라질 수 있어, 이 문서에서는 “다음/이전 함수·인자 이동이 가능하다”는 흐름만 유지한다.

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
| `"+yaf` / `"+yif` | 함수 전체/본문을 시스템 클립보드로 복사 | Normal |
| `daa` / `cia` | 인자 삭제 / 인자 교체 입력 | Normal |

### 클립보드 복사 (`"+` 레지스터)

| 키 | 동작 | 모드 |
|---|---|---|
| `"+yiw` | 단어를 시스템 클립보드로 복사 | Normal |
| `"+yy` | 현재 줄을 시스템 클립보드로 복사 | Normal |
| Visual 선택 후 `"+y` | 선택 영역을 시스템 클립보드로 복사 | Visual |
| blockwise Visual(`<C-v>`) 후 `"+y` | 직사각형 블록을 시스템 클립보드로 복사 | Visual Block |

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

정의/참조로 이동한 뒤에는 `<C-o>` / `<C-i>`를 브라우저의 뒤로/앞으로 가기처럼 써서 원래 편집 위치를 빠르게 왕복한다.

| 키 | 동작 | 모드 |
|---|---|---|
| `gd` / `gD` / `gr` / `gi` | 정의/선언/참조/구현 이동 | Normal |
| `<C-o>` / `<C-i>` | 점프 이전/다음 위치로 이동 (`gd`/`gr` 뒤로 가기·앞으로 가기) | Normal |
| `K` | hover 문서 | Normal |
| `<leader>rn` | 심볼 리네임 | Normal |
| `<leader>ca` | code action | Normal, Visual |
| `[d` / `]d` | 이전/다음 진단 | Normal |
| `<leader>le` | 현재 줄 진단 팝업 | Normal |
| `<leader>lf` | 파일 포맷팅(async) | Normal |
