# diff-view

> 🌎 [English](README.md)

Uma skill do [Claude Code](https://claude.com/claude-code) que renderiza as alterações do seu repositório git como um diff HTML lado a lado e abre num portal do [Maestri](https://www.themaestri.app/) ao lado do seu terminal.

Powered by [diff2html-cli](https://github.com/rtfpessoa/diff2html-cli), executado pelo [Bun](https://bun.com/) via `bunx --bun`.

## O que ela faz

Em vez de você ficar caçando alteração no `git diff` do terminal, é só pedir um diff visual pro Claude Code. A skill:

1. Decide o escopo certo (working tree vs `HEAD` por padrão, mas respeita "compara com a main", "último commit", "só o staged" etc.).
2. Joga isso no `bunx --bun diff2html-cli` e gera uma página HTML autocontida com layout lado a lado.
3. Inclui **arquivos untracked** também, alimentando `git diff --no-index /dev/null <arquivo>` pra cada um — sem mexer no seu index.
4. Remove o header promocional do `diff2html` usando o flag nativo `-t "git diff"`.
5. Abre o resultado num portal do Maestri chamado **Diff** (reaproveita o portal nas próximas rodadas, com uma query string de cache-bust pra forçar o reload do conteúdo novo).

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
{
  git diff HEAD
  git ls-files --others --exclude-standard | while IFS= read -r f; do
    git diff --no-index --binary -- /dev/null "$f" || true
  done
} | bunx --bun diff2html-cli -i stdin -s side -t "git diff" -F /tmp/claude-diff.html

maestri portal create "file:///tmp/claude-diff.html" "Diff"
# ou, se o portal Diff já existir:
maestri portal navigate "Diff" "file:///tmp/claude-diff.html?$(date +%s)"
```

A query string de cache-bust força o portal a recarregar, já que o caminho do arquivo não muda entre as rodadas.

## Integração com Maestri

O HTML é totalmente autocontido (CSS + JS inline) e servido via `file://`, sem precisar de servidor local. A skill mantém um único portal chamado **Diff** e aponta ele pro HTML mais recente a cada run, então a visualização do diff fica num lugar previsível na sua canvas.

## Licença

MIT
