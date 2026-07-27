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
```

Na próxima sessão do Claude Code, invoque com `/despachar <pedidos>`.

## /state — o painel da mesa

Skill companheira: `/state` renderiza um painel denso (no estilo do `/usage`)
com o que está **em voo**, **em review**, **travado esperando você** e
**concluído hoje** — cruzando o contexto da sessão, o board de tarefas
(registro definitivo) e os PRs abertos via `gh`, apontando divergências
entre as fontes em vez de escondê-las.

## Configuração

A skill assume um board de tarefas acessível por CLI (o autor usa uma skill de
Trello). Edite a seção **3. Despachar** do `SKILL.md` e troque o bloco de
registro pelo comando do seu board. Sem board, um `DESPACHOS.md` versionado no
projeto cumpre o papel de registro.

Ajuste também os "combinados" da seção **3.5** (política de merge pós-review)
para o acordo do seu time.

## Requisitos

- Claude Code com o **Agent tool** habilitado (subagentes `executor` /
  `code-reviewer` ou equivalentes).
- `gh` CLI autenticado, para PRs.
- (Opcional) CLI do seu board de tarefas.

## Por que funciona

A separação escrever/revisar em agentes distintos pega o que passa batido numa
sessão só: em produção, o review independente já segurou rollback acidental de
ponteiro de submódulo, migration com head divergente que quebraria o banco de
todos os devs, e suíte de testes que só ficava vermelha quando os arquivos
rodavam juntos.

## Licença

MIT
