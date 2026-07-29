---
name: trello-delegacao
description: Modelo de gestão via board de tarefas — preparar cards com spec completa (Definition of Ready), delegar a agentes executores e acompanhar o ciclo. Usar quando o usuário falar em "preparar card", "delegar card", "executar card #N", "card pronto pra agente" ou "status das delegações".
---

# Board de tarefas como fila de delegação para agentes

## Conceito

O board pessoal do usuário é a fonte de verdade do trabalho dele.
O modelo tem dois papéis:

- **GESTOR** (a sessão com o usuário): enriquece cards até estarem prontos para
  delegação, delega, acompanha e revisa. **Não executa** a tarefa do card.
- **EXECUTOR** (subagente, outra sessão Claude Code ou agente remoto): pega um
  card pronto, executa exatamente a spec, e devolve o resultado NO card.

A tese: se o card tem informação suficiente, qualquer agente chega "com tudo
pronto e definido" e entrega sem ida-e-volta. O trabalho do gestor é garantir isso.

Esta skill cuida de cards que **já existem** no board com spec própria. Para
trabalho que chega pela conversa, usar a skill irmã `despachar`.

## CLI

> **Configuração:** esta skill assume um board acessível por CLI (ex.: Trello).
> Ajuste os comandos abaixo para o seu board — a interface esperada está no
> README do repositório.

```bash
cd <caminho-da-skill-trello>
TRELLO_BOARD=<board> python trello <cmd>
```

## Ciclo de vida (labels no board)

| Label | Cor | Significado |
|---|---|---|
| (sem label 🤖) | — | rascunho — falta informação |
| 🤖 Pronto p/ agente | azul | spec completa, pode delegar |
| 🤖 Executando | roxo | agente trabalhando |
| 🤖 Revisar | laranja | agente terminou, aguarda revisão do usuário |

Prioridade segue nas labels "Prioridade Alta/Média/Baixa".
Colunas: `A fazer` → `Em andamento` (ao delegar) → `Concluído` (revisão aprovada).

## Modo GESTOR — "prepara o card #N"

1. Ler o card inteiro: `trello card N` (descrição, comentários, checklist).
2. Coletar contexto: código/repo citado, documentos que o usuário compartilhar,
   memória persistente, cards relacionados.
3. Preencher a spec (template abaixo) na **descrição** do card — preservar a
   linha "Origem: card #N..." se existir. Critérios de aceite viram **checklist**
   do card (checklist "Critérios de aceite").
4. O que não souber, **perguntar ao usuário** — nunca inventar decisão de negócio.
   Listar as perguntas abertas num comentário do card se ficarem pendentes.
5. Quando completo: aplicar label "🤖 Pronto p/ agente" e avisar o usuário.

### Template de spec (Definition of Ready)

```markdown
## Objetivo
(1–2 frases: o que deve existir quando terminar)

## Contexto
- Repo/worktree e branch base:
- Módulo:
- Docs/refs: (paths, links, cards relacionados)
- Decisões já tomadas: (com o porquê)

## Escopo
Inclui: ...
Fora do escopo: (explícito — é o que evita o agente "viajar")

## Critérios de aceite
(cada item vira um item do checklist do card)

## Como verificar
(comandos, testes, tela — evidência objetiva de que funcionou)

## Restrições
- Regras do repo (CLAUDE.md do projeto): branch + PR obrigatório, sem
  credencial hardcoded, conventional commits
- (restrições específicas do card)
```

## Modo GESTOR — "delega o card #N"

1. **Pré-check**: descrição tem todas as seções da spec? Tem checklist de
   critérios? Label "🤖 Pronto p/ agente" presente? Se não → voltar ao preparar.
2. Comentar no card: `🤖 Delegado em <data> — <como/para quem>`.
3. Trocar label para "🤖 Executando" e mover para "Em andamento".
4. Lançar o executor com o **prompt de delegação** abaixo:
   - Tarefa média → subagente na própria sessão (Agent tool, tipo `executor`
     ou `general-purpose`).
   - Tarefa grande → instruir o usuário a abrir sessão Claude Code dedicada
     (na worktree certa) com: `executa o card #N do meu board`.

### Prompt de delegação (injetar o conteúdo do card)

```
Você é um agente EXECUTOR. Sua tarefa é o card do board abaixo — a spec é um
contrato: execute exatamente o que ela define, escopo fechado, sem expandir.

<título, descrição e checklist do card #N>

Regras:
- Trabalhe em branch própria; entrega é via PR (nunca merge direto na main).
- Sem credencial hardcoded, conventional commits.
- Dúvida de NEGÓCIO: não decida — registre a pergunta como comentário no card e pare.
- Ao terminar: rode a verificação da seção "Como verificar", comente o resultado
  no card (resumo + link do PR) via:
  cd <caminho-da-skill-trello> &&
  TRELLO_BOARD=<board> python trello comment N "..."
  e troque a label do card para "🤖 Revisar".
```

## Modo EXECUTOR — "executa o card #N"

Quando o usuário (ou um gestor) pedir para executar um card: ler o card via CLI,
seguir a spec como contrato, comentar início e fim no card, entregar PR, marcar
itens do checklist conforme concluir, trocar label para "🤖 Revisar" ao final.
Não expandir escopo. Dúvida de negócio → comentário no card + perguntar.

## Modo GESTOR — "status das delegações"

1. Listar cards com labels 🤖 (`trello cards` + filtrar).
2. Para cada um: últimos comentários (`trello card N`) e movimentação
   (`trello history N`); PRs relacionados (`gh pr list`).
3. Resumir: o que executa, o que espera revisão, o que está travado em pergunta.

## Regras duras

- **Nunca delegar card sem spec completa** — agente perdido custa mais que perguntar antes.
- **Um card = uma entrega revisável.** Ficou grande? Quebrar em cards filhos
  ("<nome> — Parte 1: ...") linkados na descrição do pai.
- **Todo resultado de agente vira comentário no card + PR** — nada termina só na conversa.
- **Nunca deletar cards.**
- Cards copiados de um board de time têm "Origem: card #N" na descrição — ao
  concluir, avaliar com o usuário se o card do time também deve ser movido/comentado.
