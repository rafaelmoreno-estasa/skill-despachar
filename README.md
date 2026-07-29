# /despachar — mesa de despacho multi-agente para Claude Code

Skill de [Claude Code](https://claude.com/claude-code) que transforma uma sessão
numa **mesa de despacho**: você joga pedidos (um ou vários, sem relação entre
si), e a sessão vira um *master* que decompõe cada pedido em tarefas com
**Definition of Done**, despacha cada tarefa para um **agente executor** com o
modelo proporcional à dificuldade, passa todo PR por **code review independente**
e registra cada entrega num board de tarefas (Trello ou equivalente).

## O ciclo

```
pedido → triagem (tarefas + DoD) → roteamento de modelo (haiku/sonnet/opus)
       → despacho paralelo → code review independente → correção (se bloqueante)
       → verificação item a item do DoD → card fechado no board
```

Princípios:

- **O master nunca executa** — só decompõe, despacha e verifica.
- **DoD checável com evidência objetiva** ("endpoint retorna 200 com payload X",
  não "melhorar X").
- **Quem escreveu nunca revisa o próprio PR** — reviewer é outro agente,
  read-only; achado bloqueante volta pro executor original.
- **Decisão de negócio nunca é inventada** — trava a tarefa e pergunta ao dono.
- **Nada termina só na conversa** — o board é o registro definitivo, com datas
  de início/término em cada card.
- **Fila contínua** — mensagens novas entram na triagem na hora; informação
  sobre tarefa em voo vai pro agente que está rodando, sem redespacho.
- **Merge ≠ deploy** — merge pode ser autorizado pós-review limpo; deploy só
  com pedido explícito.

## Instalação

```bash
# copie as pastas das skills para o diretório de skills do Claude Code:
cp -r despachar ~/.claude/skills/despachar
cp -r state ~/.claude/skills/state
cp -r trello-delegacao ~/.claude/skills/trello-delegacao
```

Na próxima sessão do Claude Code, invoque com `/despachar <pedidos>`.

## /state — o painel da mesa

Skill companheira: `/state` renderiza um painel denso (no estilo do `/usage`)
com o que está **em voo**, **em review**, **travado esperando você** e
**concluído hoje** — cruzando o contexto da sessão, o board de tarefas
(registro definitivo) e os PRs abertos via `gh`, apontando divergências
entre as fontes em vez de escondê-las.

## /trello-delegacao — cards que já existem no board

Skill irmã para o outro lado do fluxo: enquanto o `/despachar` cuida de
trabalho que **chega pela conversa**, a `trello-delegacao` cuida de cards que
**já existem** no board. Define os dois papéis (GESTOR prepara e delega,
EXECUTOR entrega no card), o template de spec (**Definition of Ready**) e o
ciclo de labels (`🤖 Pronto p/ agente` → `🤖 Executando` → `🤖 Revisar`).
Regra central: nunca delegar card sem spec completa.

## Configuração

A skill assume um board de tarefas acessível por CLI (o autor usa uma skill de
Trello). Edite a seção **3. Despachar** do `SKILL.md` e troque o bloco de
registro pelo comando do seu board. Sem board, um `DESPACHOS.md` versionado no
projeto cumpre o papel de registro.

Ajuste também os "combinados" da seção **3.5** (política de merge pós-review)
para o acordo do seu time.

### Interface esperada do CLI de board

As skills chamam o CLI com estes comandos — qualquer board serve, desde que o
seu CLI (ou um wrapper) exponha o equivalente:

| Comando | Usado por | Para quê |
|---|---|---|
| `create "<título>" --list "<coluna>" --label "<label>" --desc "<texto>"` | despachar | registrar entrega nova |
| `comment <id> "<texto>"` | despachar, trello-delegacao | resultado/andamento em card existente |
| `done <id> "<resumo>"` | despachar | fechar card verificado (mover p/ Concluído) |
| `cards "<coluna>"` | state, trello-delegacao | listar trabalho aberto |
| `card <id>` | trello-delegacao | ler descrição, comentários e checklist |
| `history <id>` | trello-delegacao | movimentações do card |

Labels usadas no ciclo: `🤖 Pronto p/ agente`, `🤖 Executando`, `🤖 Revisar`
(+ `Prioridade Alta/Média/Baixa`). Colunas: `A fazer`, `Em andamento`,
`Concluído`.

## Requisitos

- Claude Code com o **Agent tool** habilitado (subagentes `executor` /
  `code-reviewer` ou equivalentes).
- `gh` CLI autenticado, para PRs.
- (Opcional) CLI do seu board de tarefas com a interface acima.

## Por que funciona

A separação escrever/revisar em agentes distintos pega o que passa batido numa
sessão só: em produção, o review independente já segurou rollback acidental de
ponteiro de submódulo, migration com head divergente que quebraria o banco de
todos os devs, e suíte de testes que só ficava vermelha quando os arquivos
rodavam juntos.

## Licença

MIT
