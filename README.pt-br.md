# diff-view

> 🌎 [English](README.md)

Uma skill do [Claude Code](https://claude.com/claude-code) que renderiza as alterações do seu repositório git como um diff HTML lado a lado e abre num portal do [Maestri](https://www.themaestri.app/) ao lado do seu terminal.

Powered by [diff2html-cli](https://github.com/rtfpessoa/diff2html-cli), executado pelo [Bun](https://bun.com/) via `bunx --bun`.

## O que ela faz

Em vez de você ficar caçando alteração no `git diff` do terminal, é só pedir um diff visual pro Claude Code. A skill:

1. Decide o escopo certo (working tree vs `HEAD` por padrão, mas respeita "compara com a main", "último commit", "só o staged" etc.).
2. Joga isso no `bunx --bun diff2html-cli` e gera uma página HTML autocontida com layout lado a lado.
3. Inclui **arquivos untracked** também, alimentando `git diff --no-index /dev/null <arquivo>` pra cada um — sem mexer no seu index.
4. Remove o header promocional do `diff2html` usando o flag nativo `-t` (o título vira `git diff: <nome-do-repo>`).
5. Abre o resultado num portal do Maestri **por projeto**, com o nome `Diff: <basename-do-repo>`. O nome do arquivo HTML e o nome do portal são derivados do basename do repo + um hash curto do path absoluto, então trabalhar em vários projetos ao mesmo tempo não atropela os diffs abertos — cada projeto ganha seu próprio portal na canvas.

## Instalação

Clone na pasta de skills do seu usuário:

```bash
git clone git@github.com:RodriguesCosta/diff-view-skill.git ~/.claude/skills/diff-view
```

Reinicie o Claude Code (ou recarregue as skills) e a skill fica disponível.

### Pré-requisitos

- `git` (você já tem).
- [`bun`](https://bun.com/) no PATH — a skill usa `bunx --bun` pra rodar o `diff2html-cli` no runtime do Bun. A primeira execução baixa o pacote; depois fica cacheado.
- CLI do [`maestri`](https://www.themaestri.app/) pra abrir o resultado na canvas. Sem ele, o HTML ainda é gerado em `/tmp/claude-diff.html` e dá pra abrir manualmente.

## Como usar

É só pedir. A skill dispara automaticamente em frases como:

- "mostra o diff" / "diff visual" / "diff view"
- "abre o diff no browser" / "abrir o diff no maestri"
- "ver alterações visualmente" / "review my changes"
- "compara main com HEAD" / "diff entre branches"

Por padrão mostra todas as alterações não commitadas (staged + unstaged + untracked) vs `HEAD`. Você pode pedir qualquer escopo: "diff entre main e feature/x", "me mostra o último commit", "só staged" etc.

## Como funciona

O pipeline central é mais ou menos assim:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_NAME=$(basename "$REPO_ROOT")
REPO_HASH=$(printf '%s' "$REPO_ROOT" | shasum | cut -c1-6)
DIFF_FILE="/tmp/claude-diff-${REPO_NAME}-${REPO_HASH}.html"
PORTAL_NAME="Diff: ${REPO_NAME}"

{
  git diff HEAD
  git ls-files --others --exclude-standard | while IFS= read -r f; do
    git diff --no-index --binary -- /dev/null "$f" || true
  done
} | bunx --bun diff2html-cli -i stdin -s side -t "git diff: $REPO_NAME" -F "$DIFF_FILE"

if maestri list 2>&1 | grep -Fq "name: \"$PORTAL_NAME\""; then
  maestri portal navigate "$PORTAL_NAME" "file://${DIFF_FILE}?$(date +%s)"
else
  maestri portal create "file://${DIFF_FILE}" "$PORTAL_NAME"
fi
```

A query string de cache-bust força o portal a recarregar, já que o caminho do arquivo não muda entre rodadas no mesmo repo. O hash no nome do arquivo impede que dois repos com o mesmo basename sobrescrevam o HTML um do outro.

## Integração com Maestri

O HTML é totalmente autocontido (CSS + JS inline) e servido via `file://`, sem precisar de servidor local. Cada projeto ganha seu próprio portal — `Diff: <basename-do-repo>` — então dá pra deixar diffs de projetos diferentes abertos lado a lado enquanto você pula entre eles.

## Licença

MIT
