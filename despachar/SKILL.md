---
name: despachar
description: Despachar trabalho para agentes executores — aceitar um ou VÁRIOS pedidos sem relação entre si numa mesma invocação, decompor cada um em tarefas com Definition of Done, disparar cada tarefa num agente com modelo proporcional à dificuldade e registrar toda entrega num board de tarefas (Trello ou equivalente). Usar quando o usuário disser "despacha", pedir para resolver algo "com agentes", ou entregar uma lista de tarefas para rodar em paralelo.
---

# Despacho de agentes

A sessão que invoca esta skill é o **master**: decompõe, despacha, verifica.
Todo trabalho de execução vai para um agente — o master nunca executa a tarefa
em si. O board de tarefas é o registro definitivo de cada entrega: o master
confere o que está pronto olhando o board, não a memória da conversa.

> **Configuração:** esta skill assume um board de tarefas acessível por CLI
> (ex.: Trello). Ajuste o comando de registro na seção 3 para o seu board.
> Sem board, mantenha um `DESPACHOS.md` no projeto como registro — o
> princípio é o mesmo: nada termina só na conversa.

Para cards que **já existem** no board com spec própria, usar a skill irmã
`trello-delegacao` (neste mesmo repositório); esta skill é para trabalho que
chega pela conversa.

**Modo sessão (fila contínua).** A partir da primeira invocação, a sessão
inteira vira uma mesa de despacho:

- Cada mensagem nova do usuário com trabalho novo entra na fila e passa pela
  triagem imediatamente — sem precisar reinvocar `/despachar` e sem esperar
  os ciclos anteriores fecharem. Pedidos independentes nunca bloqueiam uns
  aos outros.
- Uma invocação pode trazer **vários pedidos sem relação entre si** (bugfix +
  feature + script, por exemplo). Separar por pedido primeiro, depois
  decompor cada pedido em tarefas; despachar tudo que estiver pronto num
  único bloco paralelo. Pedido que depende de decisão de negócio fica retido
  na fila — os demais seguem.
- Informação nova sobre tarefa **já em voo** (print, erro exato, contexto
  adicional) não vira despacho novo: encaminhar ao agente que está rodando
  via SendMessage. Despacho novo só para trabalho novo.
- Ao reportar, sempre incluir o estado da fila inteira: o que acabou de ser
  despachado, o que segue em voo, o que está travado esperando o usuário.

## 1. Triagem — tarefas + Definition of Done

Decompor cada pedido em tarefas independentes (uma tarefa = uma entrega
revisável). Para cada tarefa, escrever a **Definition of Done (DoD)**: itens
checáveis com evidência objetiva — comando + saída esperada, PR aberto, teste
passando, tela funcionando. "Melhorar X" não é DoD; "endpoint /y retorna 200
com o payload Z" é.

Decisão de negócio em aberto → perguntar ao usuário antes de despachar, nunca
inventar.

**Pronto quando:** toda tarefa tem DoD checável e nenhuma pergunta de negócio
pendente. Apresentar a lista `tarefa → modelo → DoD` (uma linha cada) e
despachar em seguida, sem pedir permissão.

## 1b. Quebra, paralelismo e quick wins

Quebrar por **independência**, não por pressa: só paraleliza o que não
compartilha arquivos nem decisões (módulos diferentes, pesquisa +
implementação, frentes sem interseção) — e cada agente que muta o repo
trabalha na sua própria worktree. Feature coesa (migration → serviço → UI no
mesmo módulo) NÃO se fatia: o custo de integrar as metades come o ganho.

**Quick wins primeiro**: se a entrega aceita corte incremental, a tarefa 1 é o
menor recorte que já gera valor em produção (PR pequeno, mergeável hoje); os
refinamentos viram tarefa 2 despachada em seguida — de preferência
**retomando o mesmo agente** (SendMessage), que na prática custa cerca de
metade de um despacho fresco por já carregar o contexto. Apresentar na triagem
qual é o quick win e o que ficou para a segunda leva.

## 2. Rotear o modelo pela dificuldade

| Dificuldade | Sinais | `model` |
|---|---|---|
| Trivial | mecânica, 1 arquivo, zero decisão (rename, ajuste de texto, rodar script) | `haiku` |
| Padrão | feature contida, bugfix com causa conhecida, script novo, CRUD | `sonnet` |
| Difícil | multi-arquivo, arquitetura, debugging sem causa, migration, código financeiro/fiscal | `opus` |

Na dúvida entre dois níveis, usar o maior.

## 3. Despachar

Agent tool, tipo `executor` (`general-purpose` apenas para pesquisa/
investigação read-only), com o `model` do passo 2. Tarefas independentes →
todos os despachos **num único bloco**, em paralelo.

**Regra dura — agentes deste repositório.** Os tipos `executor` e
`code-reviewer` referem-se às definições em `agents/` deste repositório,
instaladas em `~/.claude/agents/`. Se o tipo não existir no ambiente,
**pare e instrua a instalação** (`cp -f agents/*.md ~/.claude/agents/`) em
vez de despachar para `general-purpose` ou outro substituto — as garantias
do fluxo (escopo fechado do executor, reviewer read-only) dependem dessas
definições.

Prompt do executor:

```
Você é um agente EXECUTOR despachado. A tarefa e o Definition of Done abaixo
são contrato: execute exatamente isso, escopo fechado, sem expandir.

## Tarefa
<descrição, contexto, paths relevantes>

## Definition of Done
<itens checáveis>

Regras:
- Branch própria + PR (nunca merge direto na main); sem credencial hardcoded;
  conventional commits.
- Dúvida de NEGÓCIO: não decida — registre a pergunta no seu retorno e pare.
- Penúltimo passo, obrigatório: registrar a entrega no board de tarefas:
    <COMANDO DO SEU BOARD — ex. com a skill trello:>
    cd <caminho-da-skill-trello>
    TRELLO_BOARD=<board> python trello create "<título natural da tarefa>" \
      --list "Em andamento" --label "🤖 Revisar" \
      --desc "<resumo + link do PR + DoD item a item com evidência + Início: dd/mm/aaaa hh:mm>"
  (Se a tarefa nasceu de um card #N existente, comentar no card em vez de
  criar um novo.)
- Todo card carrega DATAS: o create registra "Início: <dd/mm hh:mm>"; ao
  fechar ou pausar, o resumo/comentário registra "Término: <dd/mm hh:mm>"
  (ou o motivo da pausa com horário). O master confere isso na verificação
  como item de DoD implícito.
- Retorno final: para cada item do DoD, a evidência que o comprova
  (comando + saída, link do PR, path do arquivo).
```

Dicas de campo (aprendidas em produção):
- **Worktrees isoladas** para tarefas paralelas no mesmo repo — e `git add`
  com paths explícitos, nunca `-A` (um `git add -A` com submódulo
  desatualizado já causou rollback acidental de ponteiro em PR).
- Primeiro passo de todo executor num repo: `git remote -v` para confirmar
  que está no repo certo.
- Tarefas paralelas que precisam de servidor de teste: cada uma na sua porta.

## 3b. Previsão de duração

Ao apresentar a triagem, estimar quanto tempo cada tarefa vai levar
consultando `historico_despachos.jsonl` (na pasta desta skill; criado no
primeiro registro): filtrar por tags parecidas (modelo, módulo, migration,
pesquisa, ui, fix, feature...) e usar a **mediana** das durações reais como
previsão — reportar como "~25 min (base: 4 parecidas)". Sem histórico
parecido → dizer "sem base histórica" e usar a classe: haiku 2–5 min ·
sonnet 5–20 · opus 20–50 · pesquisa/discovery 5–15.

Ao receber o retorno (a task-notification traz `duration_ms` e tokens),
**antes de fechar o card**, registrar UMA linha no jsonl via append:

```json
{"data":"2026-08-11","titulo":"...","tipo":"executor|pesquisa|review|auditoria","model":"opus","tags":["modulo","migration"],"previsto_min":25,"real_min":44,"tokens":193472,"card":"#132","nota":"por que desviou, se desviou"}
```

Previsto × real com desvio grande (>2x para qualquer lado) → citar no delta
com a causa ("subestimei: tinha migration escondida") — é assim que a
previsão calibra. O histórico é da mesa, não do projeto: nunca commitar no
repositório do projeto.

## 3.5 Code review automático dos PRs

Toda tarefa cuja entrega inclui um PR passa por review antes de fechar o
card: quando o executor retorna, o master despacha um agente `code-reviewer`
(read-only) com o link do PR — `opus` para código financeiro/fiscal,
migrations ou submódulos; `sonnet` para o resto. Quem escreveu nunca revisa
o próprio PR.

- Achado bloqueante → redespachar o executor original citando os achados.
- Review limpo → seguir o combinado vigente com o usuário sobre merge
  (sugestão: merge autorizado após review limpo; deploy NUNCA automático).
  PR de submódulo mergeado exige bump do ponteiro no superprojeto (branch +
  PR, checando conflict markers no commit-alvo antes).
- O resultado do review entra no card do board.
- Merge não é deploy: deploy só com pedido explícito do usuário.

## 4. Verificar e fechar o ciclo

Quando os agentes retornarem, para cada tarefa conferir **item a item do DoD**
contra a evidência apresentada:

- DoD cumprido → confirmar o card no board — o card concluído é o registro
  definitivo.
- Item de DoD sem evidência → redespachar o mesmo tipo de agente nomeando
  exatamente o item faltante.
- Agente terminou sem registrar no board → o master cria o card antes de
  reportar; nada termina só na conversa.

**Pronto quando:** cada tarefa está (a) verificada e com card concluído, ou
(b) redespachada com o item faltante nomeado, ou (c) travada em pergunta de
negócio registrada. Reportar ao usuário nesses três grupos. Havendo mais
trabalho na fila, voltar ao passo 1 — o ciclo de despacho não para enquanto
houver tarefa sem card.
