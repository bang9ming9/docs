# Neovim 사용 흐름 메모

이 문서는 Neovim에서 실제로 자주 사용하는 작업 흐름을 정리한 메모다. 모든 Vim 키를 빠짐없이 나열하기보다, 코드를 읽고 수정하고 문서로 옮기는 과정에서 반복해서 쓰는 조작을 중심으로 정리한다.

현재 주로 사용하는 흐름은 다음과 같다.

- 함수나 타입 정의로 이동한 뒤 다시 원래 위치로 돌아오기
- Git diff로 변경사항 확인하기
- 프로젝트 안에서 코드 검색하기
- 코드 조각을 시스템 클립보드로 복사하고 붙여넣기
- Markdown 문서를 작성하면서 preview 확인하기
- 같은 단어를 여러 개 선택해 동시에 수정하기

`<leader>`는 `\`로 사용한다.

## Vim 키를 외우기보다 조립하기

Vim 키는 보통 `동작 + 대상` 구조로 조합된다. 모든 단축키를 하나씩 외우기보다, 몇 가지 기본 규칙을 이해하면 필요한 조작을 직접 조립해서 떠올릴 수 있다.

| 키 | 의미 | 기억 방식 |
| --- | --- | --- |
| `d` | 삭제 | delete |
| `y` | 복사 | yank |
| `c` | 변경 | change |
| `p` | 붙여넣기 | put |
| `g` | 이동 계열 접두어 | go |
| `i` | 안쪽 | inner |
| `a` | 주변 포함 | around |

예를 들어 `ciw`는 `change inner word`로 읽을 수 있다. 현재 단어 안쪽을 지우고 입력 모드로 들어가는 조작이다.

`"+yaf`도 같은 방식으로 나눠서 읽을 수 있다.

```text
"+   = 시스템 클립보드 레지스터
y    = yank
af   = around function
```

즉 “함수 전체를 시스템 클립보드로 복사한다”는 뜻이다. 이렇게 보면 긴 키도 무작정 외우는 대상이 아니라 조합 가능한 문장에 가까워진다.

## 정의를 확인하고 다시 돌아오기

코드를 읽을 때 가장 자주 쓰는 흐름은 호출부에서 정의로 들어갔다가 다시 원래 자리로 돌아오는 것이다.

| 키 | 동작 | 기억 방식 |
| --- | --- | --- |
| `gd` | 정의로 이동 | go definition |
| `gD` | 선언으로 이동 | go declaration |
| `gr` | 참조 찾기 | go references |
| `gi` | 구현으로 이동 | go implementation |
| `K` | hover 문서 확인 | 도움말/문서 확인 관습 |
| `Ctrl+o` | 이전 점프 위치로 돌아가기 | old position |
| `Ctrl+i` | 다시 다음 점프 위치로 이동 | jump list forward |
| `` `. `` | 마지막 수정 위치로 이동 | last change |
| `Ctrl+^` | 직전 버퍼로 전환 | alternate file |

예를 들어 함수 호출부에서 `gd`로 구현을 확인한 뒤, `Ctrl+o`로 다시 호출부로 돌아온다. 이 흐름을 알아야 정의 이동을 부담 없이 사용할 수 있다.

`Ctrl+o`와 `Ctrl+i`는 LSP 전용 키가 아니라 Vim의 jump list를 사용하는 키다. 따라서 `gd`뿐 아니라 검색, mark 이동, 파일 내 큰 이동 이후에도 이전 위치로 돌아갈 때 사용할 수 있다.

반면 `Ctrl+^`는 “이전 점프 위치”가 아니라 “직전에 보던 버퍼”로 전환한다. 파일 A에서 파일 B로 이동한 뒤 다시 A로 빠르게 돌아가고 싶을 때 유용하다.

## 현재 파일에서 같은 단어 빠르게 찾기

프로젝트 전체 검색은 Telescope가 편하지만, 현재 파일 안에서 같은 단어를 찾을 때는 Vim 기본 검색이 더 빠를 때가 많다.

| 키 | 동작 | 기억 방식 |
| --- | --- | --- |
| `*` | 커서 위 단어를 아래 방향으로 검색 | same word |
| `#` | 커서 위 단어를 위 방향으로 검색 | `*`의 반대 |
| `n` | 다음 검색 결과 | next |
| `N` | 이전 검색 결과 | next의 반대 |
| `:noh` | 검색 하이라이트 끄기 | no highlight |

예를 들어 변수명 위에 커서를 두고 `*`를 누르면 현재 파일에서 같은 단어를 바로 검색한다. 단순히 “이 이름이 이 파일에서 또 어디 쓰였는지” 확인하는 정도라면 `\fg`보다 가볍다.

## Git diff 확인하기

변경사항 확인은 범위에 따라 도구를 나눠서 사용한다. 작업 중인 파일의 작은 변경은 gitsigns로 보고, 브랜치나 PR 단위 변경은 diffview로 본다.

### 현재 파일의 변경 확인

| 키 | 동작 | 기억 방식 |
| --- | --- | --- |
| `]h` | 다음 변경 블록으로 이동 | next hunk |
| `[h` | 이전 변경 블록으로 이동 | previous hunk |
| `\hp` | 변경 내용 미리보기 | hunk preview |
| `\hb` | 현재 줄 blame | hunk blame |
| `\hd` | 현재 파일 diff | hunk diff |

작은 수정은 `]h`, `[h`로 이동하면서 확인한다. 특정 줄이 왜 존재하는지 궁금하면 `\hb`로 blame을 확인한다.

### 브랜치나 PR 단위 diff 확인

| 키 | 동작 | 기억 방식 |
| --- | --- | --- |
| `\do` | diff view 열기 | diff open |
| `\dc` | diff view 닫기 | diff close |
| `\dh` | 현재 파일 히스토리 | diff history |
| `\dH` | 프로젝트 전체 히스토리 | History, 대문자는 더 넓은 범위 |
| `\dm` | main/master 기준 비교 | diff main/master |
| `\dB` | 두 브랜치 비교 | diff branches |
| `]c` | 다음 변경 블록 | next change |
| `[c` | 이전 변경 블록 | previous change |

Diffview 안에서는 `{`와 `}`보다 `]c`, `[c`를 쓰는 편이 낫다. `{`, `}`는 문단 이동이고, `]c`, `[c`는 diff의 변경 블록 단위 이동이기 때문이다.

GitHub PR처럼 base 브랜치와 target 브랜치를 비교하려면 `\dB`를 사용한다. 이때 내부적으로는 `base...target` 형태의 triple-dot 비교를 사용하는 흐름으로 이해하면 된다.

## 코드 검색하기

코드 검색은 Telescope를 사용한다. 파일명을 알고 있는지, 문자열을 알고 있는지에 따라 키를 나눠 쓰면 된다.

| 키 | 동작 | 기억 방식 |
| --- | --- | --- |
| `\ff` | 파일명 검색 | find files |
| `\fg` | 프로젝트 전체 문자열 검색 | find grep |
| `\fb` | 열린 버퍼 검색 | find buffers |
| `\fr` | 최근 파일 검색 | find recent |
| `\fs` | 현재 파일 심볼 검색 | find symbols |
| `\fS` | 프로젝트 전체 심볼 검색 | Symbols, 대문자는 더 넓은 범위 |
| `\fd` | 진단 목록 검색 | find diagnostics |

보통은 다음 기준으로 사용한다.

- 파일 이름이 기억나면 `\ff`
- 함수명, 에러 메시지, 설정 키워드를 찾고 싶으면 `\fg`
- 현재 파일의 함수 목록을 보고 싶으면 `\fs`
- 프로젝트 전체의 심볼을 찾고 싶으면 `\fS`

## 복사와 붙여넣기

코드를 외부 문서, 브라우저, ChatGPT, Slack 등에 붙여넣을 때는 시스템 클립보드 레지스터를 사용한다.

Vim의 기본 yank는 Vim 내부 레지스터에만 들어간다. 외부 앱과 공유하려면 `"+`를 앞에 붙인다.

| 키 | 동작 | 읽는 방법 |
| --- | --- | --- |
| `yy` | 현재 줄을 Vim 내부에 복사 | yank line |
| `"+yy` | 현재 줄을 시스템 클립보드에 복사 | yank line to clipboard |
| `"+yiw` | 현재 단어를 시스템 클립보드에 복사 | yank inner word |
| `"+yip` | 현재 문단을 시스템 클립보드에 복사 | yank inner paragraph |
| `"+p` | 시스템 클립보드에서 붙여넣기 | put from clipboard |

Treesitter textobjects를 사용하면 함수 단위 복사도 할 수 있다.

| 키 | 동작 | 읽는 방법 |
| --- | --- | --- |
| `"+yaf` | 함수 전체를 시스템 클립보드에 복사 | yank around function |
| `"+yif` | 함수 본문만 시스템 클립보드에 복사 | yank inside function |
| `"+yac` | struct/interface/class 전체 복사 | yank around class |
| `"+yia` | 인자 값만 복사 | yank inside argument |

`"+yaf`는 처음에는 길어 보이지만, 구조를 나누면 단순하다.

```text
"+   = 시스템 클립보드
y    = yank
af   = around function
```

즉 “함수 전체를 시스템 클립보드로 복사한다”는 뜻이다.

## 붙여넣기 중 레지스터를 덮지 않기

복사해둔 값을 여러 곳에 붙여넣고 싶은데, 중간에 삭제나 변경을 하면 기본 레지스터가 덮이는 경우가 있다. 이때는 black-hole register를 사용할 수 있다.

| 키 | 동작 |
| --- | --- |
| `"_d` | 삭제하지만 기본 레지스터를 덮지 않음 |
| `"_c` | 변경하지만 기본 레지스터를 덮지 않음 |
| Visual 선택 후 `"_dP` | 기존 yank 값을 유지한 채 선택 영역 교체 |

이 키는 처음부터 외울 필요는 없지만, 같은 값을 여러 군데 붙여넣는 작업이 많아질 때 유용하다.

## Visual 선택 복원과 문단 단위 조작

문서 작성이나 코드 블록 정리 중에는 방금 선택한 영역을 다시 선택하거나, 문단 단위로 조작하는 일이 자주 있다.

| 키 | 동작 | 기억 방식 |
| --- | --- | --- |
| `gv` | 마지막 Visual 선택 영역 복원 | go visual |
| `vip` | 현재 문단 선택 | visual inner paragraph |
| `yip` | 현재 문단 복사 | yank inner paragraph |
| `dip` | 현재 문단 삭제 | delete inner paragraph |
| `>` / `<` | 선택 영역 들여쓰기/내어쓰기 | 방향 기호 |
| `=` | 선택 영역 자동 정렬 | indent/format |

`gv`는 선택 영역을 조작한 뒤 다시 같은 영역을 선택할 때 좋다. 예를 들어 visual mode에서 한 번 들여쓰기한 뒤 같은 영역을 다시 들여쓰기하거나, 복사 후 같은 범위를 다시 편집할 때 사용할 수 있다.

## 같은 단어를 여러 개 선택하기

VSCode의 `Cmd+D`처럼 같은 단어를 하나씩 추가 선택하는 동작은 Vim 기본 기능보다는 multi-cursor 플러그인 영역에 가깝다.

기본 Vim에서는 목적에 따라 다음처럼 나눠서 처리한다.

| 목적 | 추천 방식 |
| --- | --- |
| 같은 단어 탐색 | `*`, `#`, `n`, `N` |
| 현재 파일 단순 치환 | `:%s/old/new/gc` |
| 프로젝트 심볼 이름 변경 | `\rn` |
| 다중 커서 편집 | `vim-visual-multi` |

단순히 이름을 바꾸는 목적이라면 LSP rename인 `\rn`이 더 안전할 때가 많다. Go처럼 LSP가 심볼 정보를 잘 알고 있는 언어에서는 문자열 치환보다 rename이 의도를 더 정확히 반영한다.

반대로 같은 단어 중 일부만 골라서 동시에 수정하고 싶다면 `vim-visual-multi`를 사용한다. 현재 설정에서는 visual-multi의 기본 흐름을 유지하기 위해 `<C-n>`을 단어 선택 키로 남기고, Neo-tree 토글은 `<leader>e`로 옮긴다.

## Markdown preview

Markdown 문서를 작성할 때는 preview를 켜서 렌더링 결과를 확인한다.

| 키 / 명령 | 동작 | 기억 방식 |
| --- | --- | --- |
| `\mp` | Markdown preview 토글 | markdown preview |
| `:MarkdownPreview` | 미리보기 시작 | 명령 이름 그대로 |
| `:MarkdownPreviewStop` | 미리보기 종료 | 명령 이름 그대로 |
| `:MarkdownPreviewToggle` | 미리보기 토글 | 명령 이름 그대로 |

표, Mermaid, 긴 목록이 들어간 문서는 Neovim 안에서만 보는 것보다 preview로 렌더링 결과를 확인하는 편이 안전하다.

## 문제가 생겼을 때 확인할 명령

동작하지 않는 키가 있을 때는 먼저 어떤 영역의 문제인지 나눠서 본다.

| 명령 | 확인 대상 |
| --- | --- |
| `:Lazy` | 플러그인 설치와 로딩 상태 |
| `:Mason` | LSP, formatter, linter 설치 상태 |
| `:LspInfo` | 현재 버퍼에 붙은 LSP |
| `:LspRestart` | LSP 재시작 |
| `:TSInstallInfo` | Treesitter parser 설치 상태 |
| `:checkhealth` | Neovim 전체 상태 |
| `:checkhealth lsp` | LSP 상태 |
| `:checkhealth nvim-treesitter` | Treesitter 상태 |

예를 들어 `gd`가 동작하지 않으면 `:LspInfo`로 LSP가 붙었는지 확인한다. `"+yaf`가 동작하지 않으면 클립보드보다 Treesitter textobjects 설정이 잡혀 있는지 먼저 확인한다. `<C-n>`이 기대와 다르게 동작한다면 Neo-tree와 visual-multi의 키 충돌이 남아 있는지 확인한다.

## 정리

이 문서는 Neovim 단축키를 많이 외우기 위한 문서가 아니다. 실제로 자주 쓰는 작업 흐름을 기준으로, 왜 그 키를 쓰는지 기억할 수 있게 정리한 메모다.

현재 기준에서 가장 중요한 흐름은 다음이다.

- `gd`로 정의를 확인하고 `Ctrl+o`로 돌아온다.
- `*`, `#`, `n`, `N`으로 현재 파일의 같은 단어를 빠르게 탐색한다.
- `\do`, `\dB`, `]c`, `[c`로 diff를 확인한다.
- `\ff`, `\fg`, `\fs`로 필요한 코드를 찾는다.
- `"+yy`, `"+yaf`로 외부에 공유할 코드를 복사한다.
- `<C-n>`으로 visual-multi 선택을 시작하고, Neo-tree는 `<leader>e`로 연다.
- `\mp`로 Markdown 렌더링을 확인한다.

이 정도 흐름만 익숙해져도 Neovim을 “설정해둔 에디터”가 아니라 실제 개발과 리뷰에 쓰는 도구로 안정적으로 사용할 수 있다.
