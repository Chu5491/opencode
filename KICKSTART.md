# KICKSTART — Chu5491/opencode 포크 빌드 가이드

이 저장소는 [anomalyco/opencode](https://github.com/anomalyco/opencode)를 포크해서
ACP 관련 패치를 얹고 **직접 빌드해서 쓰는** 용도다. 새 머신에서 세팅할 때 이 문서만 따라가면 된다.

## 1. upstream 과 다른 점

| 커밋 | 내용 |
|---|---|
| `5edc599f3` | question 툴을 ACP `elicitation/create` 로 브리지 (upstream PR #42158) |
| `6ac7b0fbd` | 클라이언트가 elicitation 미지원(JetBrains 등)이면 `session/request_permission` 으로 선택지 표시. 단일 선택만 가능, 자유 입력 불가 |
| `871feaada` | `OPENCODE_CLIENT=acp` 일 때도 question 툴 노출. 환경변수 `OPENCODE_ENABLE_QUESTION_TOOL` 불필요 |
| `e321000d4` | `.husky/pre-push` 가 `packages/opencode` 만 typecheck (전체 typecheck 는 다른 패키지의 환경 의존 오류로 실패) |

기본 브랜치는 `dev`. upstream 릴리스 태그를 주기적으로 여기에 머지한다.

## 2. 사전 준비

- bun `^1.3.14` (루트 `package.json` 의 `packageManager` 기준. 빌드 스크립트가 버전을 검사한다)
- python3 (버전 문자열 추출용, macOS 기본 포함)

```bash
git clone https://github.com/Chu5491/opencode.git ~/.local/src/opencode
cd ~/.local/src/opencode
git remote rename origin fork
git remote add origin https://github.com/anomalyco/opencode.git
```

> remote 이름 규약: `origin` = upstream(anomalyco), `fork` = Chu5491.
> clone 직후에는 `origin` 이 fork 를 가리키므로 위처럼 바꿔 둔다. 이 문서의 명령은 모두 이 규약을 따른다.

## 3. 빌드

```bash
cd ~/.local/src/opencode
bun install

cd packages/opencode
OPENCODE_VERSION=$(python3 -c "import json;print(json.load(open('package.json'))['version'])") \
  bun run script/build.ts --single --skip-install
```

- `--single`: 현재 플랫폼용 바이너리 하나만 만든다.
- `OPENCODE_VERSION` 을 안 주면 브랜치 이름이 `latest` 가 아니라서 `0.0.0-dev-…` 로 찍히고, 플러그인 버전 검사에 걸린다.
- 결과: `packages/opencode/dist/opencode-darwin-arm64/bin/opencode` (Intel Mac 은 `darwin-x64`)

바이너리 연결:

```bash
ln -sf ~/.local/src/opencode/packages/opencode/dist/opencode-darwin-arm64/bin/opencode ~/.local/bin/opencode
opencode --version   # upstream 버전 문자열 그대로 나와야 함
```

## 4. 클라이언트 연결

바이너리를 `opencode acp` 로 띄우면 된다. 환경변수는 필요 없다.

### Zed — `~/.config/zed/settings.json`
```json
"agent_servers": {
  "OpenCode": {
    "type": "custom",
    "command": "/Users/<me>/.local/bin/opencode",
    "args": ["acp"]
  }
}
```
elicitation 지원 → 선택지 + 자유 입력 모두 동작.

### JetBrains — `~/.jetbrains/acp.json`
```json
{
  "agent_servers": {
    "OpenCode": {
      "command": "/Users/<me>/.local/bin/opencode",
      "args": ["acp"]
    }
  }
}
```
elicitation 미지원 → permission 다이얼로그로 선택지만 표시.

### Devin Desktop
PATH 에서 `opencode` 를 찾아 `opencode acp` 로 띄운다. `~/.local/bin` 이 PATH 에 있으면 끝.

## 5. upstream 릴리스 따라가기

```bash
cd ~/.local/src/opencode
git fetch origin --tags
git merge --no-edit v1.18.31        # 원하는 릴리스 태그
```

충돌은 거의 항상 `bun.lock` 과 각 패키지 `package.json` 의 `"version"` 줄이다.
코드 충돌이 아닌지 확인한 뒤 upstream 쪽을 채택한다:

```bash
for f in $(git diff --name-only --diff-filter=U); do git checkout --theirs -- "$f" && git add -- "$f"; done
git commit --no-edit
```

이후 3 번 빌드 절차를 다시 돌리고, 아래 검증을 거쳐 push 한다:

```bash
cd packages/opencode
bun test --timeout 30000 test/acp/elicitation.test.ts test/cli/acp/elicitation.test.ts
cd ../..
git push fork HEAD:dev
```

## 6. 검증

- `opencode --version` 이 upstream 릴리스 번호와 같다.
- `OPENCODE_CLIENT=acp opencode serve --port 4096` 을 띄우고
  `curl localhost:4096/experimental/tool/ids` 결과에 `question` 이 있다.
- 클라이언트를 다시 연결하기 전에 옛 프로세스를 정리한다: `pkill -f "opencode acp"`

## 7. 알려진 한계

- JetBrains 에서는 자유 입력 질문이 불가. 클라이언트 쪽 elicitation 지원이 필요하다.
- upstream PR #42158 이 머지되면 패치 커밋을 버리고 정식 릴리스로 갈아탄다.
- pre-push 가 `packages/opencode` 만 검사하므로 다른 패키지를 건드리는 머지는 CI 를 따로 보지 않는다.
- `~/.bun/install/cache` 에 root 소유 파일이 섞여 install 이 실패하면
  `BUN_INSTALL_CACHE_DIR=~/.bun/install/cache-local` 로 우회한다.
