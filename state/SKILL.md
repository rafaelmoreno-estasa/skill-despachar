---
name: state
description: Painel de estado da mesa de despacho — mostra num relance o que está em voo, em review, travado esperando o usuário e concluído, no estilo do /usage. Usar quando o usuário digitar /state, perguntar "como estão as coisas", "status da fila", "o que está rodando" ou pedir o estado dos despachos.
---

# /state — painel da mesa de despacho

Renderizar um painel compacto do estado atual do trabalho despachado. Fontes,
nesta ordem (usar todas as disponíveis):

1. **Contexto da sessão** — agentes em voo desta conversa (executores,
   reviewers), com a tarefa de cada um. Só a sessão atual conhece isso; se a
   sessão for nova e não houver contexto, pular sem inventar.
2. **Board de tarefas (registro definitivo)** — os cards de trabalho aberto
   e os concluídos de hoje. Ajuste o comando ao seu board (ex. com uma skill
   de Trello):
   ```bash
   cd <caminho-da-skill-trello>
   TRELLO_BOARD=<board> python trello list
   ```
   O board vence a memória da conversa em caso de conflito.
3. **PRs abertos** — para cada repo que aparecer nos cards/contexto:
   ```bash
   gh pr list --repo <owner/repo> --state open --json number,title,headRefName
   ```

## Formato de saída (imitar a densidade do /usage)

```
📊 Mesa de despacho — <data hora>

🚀 EM VOO (<n>)
  <tarefa> · <executor|reviewer> · <desde hh:mm> · card #N

🔍 EM REVIEW (<n>)
  PR #N <repo> — <título curto> · rodada <1ª|re-review>

⛔ TRAVADO ESPERANDO VOCÊ (<n>)
  <item> — <o que falta decidir/fazer>

✅ CONCLUÍDO HOJE (<n>)
  PR #N mergeado — <o quê> · card #N fechado

📋 FILA (ainda não despachado)
  <item>
```

Regras:
- Uma linha por item; sem parágrafos. Truncar títulos em ~60 chars.
- "Travado esperando você" lista APENAS itens cuja próxima ação é do usuário
  (decisão de negócio, ação manual em console externo, rotulagem) — nunca
  itens que um agente resolve.
- Se as fontes divergirem (card fechado mas PR aberto, agente terminou sem
  card), apontar a divergência numa linha `⚠` em vez de esconder.
- Zero itens numa seção → omitir a seção.
- Terminar com uma linha-resumo: `Σ <voo> em voo · <rev> em review · <trav>
  com você · <done> fechados hoje`.
- Não despachar nada novo a partir do /state — é painel, não gatilho.
