# Refactor da Lógica de Estados e Reconciliação

**Data:** 18 Abril 2026
**Contexto:** Sessão de desenho (não implementação). Consolida o modelo final de estados e transições do MediCheck, identifica os gaps entre o modelo e o código actual (`index.html` ~2354 linhas), e prepara prompts para executar em Claude Code.

**Regras permanentes:** Só modificar `index.html`. Commits em português. Implementação em Claude Code; esta nota é o briefing.

---

## 1. Modelo de Estados

Cinco estados. Mutuamente exclusivos.

| Estado | Significado | Tipo |
|---|---|---|
| `registado` | Acto introduzido na app. Ainda dentro do prazo razoável de pagamento. | Inicial |
| `em_falta` | Passaram 3 meses do acto sem pagamento da CUF. Candidato a reclamação. | Intermédio |
| `reclamado` | Incluído em relatório enviado à CUF. À espera de resposta. | Intermédio |
| `pago` | CUF pagou. Match com fatura mensal ou marcação manual. | Terminal (feliz) |
| `rejeitado` | CUF respondeu negativamente à reclamação. Não vai ser pago. | Terminal (perdido) |

---

## 2. Transições

### Válidas (fluxo normal)

| De | Para | Trigger | Automático? |
|---|---|---|---|
| — | `registado` | User cria acto | — |
| `registado` | `em_falta` | 3 meses desde data do acto, sem pagamento | Sim (clock temporal) |
| `registado` | `pago` | Match na fatura CUF (terceira via) | Sim (reconciliação) |
| `em_falta` | `reclamado` | Botão após gerar e copiar relatório | Manual |
| `em_falta` | `pago` | Match na fatura CUF (CUF pagou tarde sem reclamação) | Sim (reconciliação) |
| `reclamado` | `pago` | Match na fatura CUF + fallback manual | Automático + manual |
| `reclamado` | `rejeitado` | Manual, com motivo | Manual |

### Reversíveis (correcção de erro humano)

| De | Para | Uso |
|---|---|---|
| `pago` | `reclamado` ou `em_falta` | User percebeu que não tinha sido pago |
| `reclamado` | `em_falta` | Desfazer marcação prematura |
| `em_falta` | `registado` | Dúvida sobre data ou correcção |
| `rejeitado` | `reclamado` | CUF reverteu decisão |

Todas as reversões exigem confirmação explícita na UI.

### Proibidas

- Criar acto directamente em qualquer estado que não seja `registado`
- `registado` → `reclamado` directo (tem de passar por `em_falta`)
- `registado` → `rejeitado` (não faz sentido)
- `pago` → `rejeitado` (não faz sentido)

### Fora do MVP (porta aberta, não abrir)

- **Bootstrap histórico** (criar directo em `pago`) — operação one-off via import XLSX, pós-MVP
- **Grupo 3 auto-populate** (linhas da fatura sem match viram actos `pago` na BD) — decisão após primeiros testes reais

---

## 3. Invariantes

1. Todo acto começa em `registado`
2. Se `status = em_falta`, então `data_acto < agora - 90 dias`
3. Se `status = reclamado`, o acto passou por `em_falta` em algum momento
4. Se `status = rejeitado`, o acto passou por `reclamado` em algum momento
5. `pago` e `rejeitado` são terminais — não revertem automaticamente (só por edição manual com confirmação)
6. Dois actos do mesmo doente na mesma data podem estar em estados diferentes (não há bundling)

---

## 4. Triggers Detalhados

### Clock temporal (gap 2)

Função `checkEmFaltaByTime()` corre ao carregar a app e quando o utilizador volta ao dashboard. Percorre `procedimentos`, e para cada acto em `registado` com `data < hoje - 90 dias`, marca como `em_falta`. Totalmente independente da reconciliação.

### Reconciliação (gaps 1 e 7)

Ao carregar fatura CUF:

1. Extrai linhas: processo + data + nome + valor
2. Cria chave `normProc(processo)|data` para cada linha
3. Para cada chave, procura actos na BD em estados `{registado, em_falta, reclamado}`. Primeiro match estrito (processo+data exactos); depois, para linhas sem match, tenta match com tolerância ±3 dias se houver candidato único.
4. Match 1:1 → marca `pago` + preenche valor + `pagoEm` + `pagoViaReconciliacao = true` (campo `procedimento` preservado — não é sobrescrito pela fatura)
5. Match n:m → UI de resolução de ambiguidade (já existe)
5.5. Matches com tolerância de datas → UI pede confirmação explícita por checkbox; nunca aplica automaticamente.
6. **Linhas da fatura sem match** → contar e mostrar nota discreta: *"N linhas da fatura sem correspondência — ignoradas"*. Não cria actos nem pede acção

### Marcar como Reclamado (gap 3)

Sequência obrigatória:

1. User abre separador "Reclamar" → vê actos `em_falta`
2. Gera texto de email → clica "Copiar email" (activa flag interna `relatorioCopiado = true`)
3. Só *após* copiado é que o botão "Confirmar envio à CUF e marcar como reclamado" fica activo
4. Ao clicar → todos os `em_falta` → `reclamado` + `reclamadoEm = now`

### Rejeitar (gap 6)

No detalhe do acto, se `status = reclamado`, botão "CUF rejeitou reclamação". Prompt com motivo:

- "Código incorrecto"
- "Doente não CUF"
- "Fora de prazo"
- "Outro" + texto livre

Ao confirmar → `rejeitado` + `rejeitadoEm = now` + `motivoRejeicao = "..."`.

---

## 5. Painel de Recuperação (novo — gap 4)

**Não é `renderRelatorio`.** Esse é um painel de casuística e deve ser renomeado para "Casuística" ou "Estatísticas".

O Painel de Recuperação mostra o **estado da operação** em euros e número:

- **Total em risco** agora: € em `em_falta` + € em `reclamado`
- **Em reclamação activa**: € em `reclamado`
- **Total recuperado**: € em `pago`. Sub-valores: este ano / acumulado.
- **Taxa de sucesso**: `pagos / (pagos + rejeitados)` — só se gap 6 estiver feito
- **Lista dos 10 casos mais antigos** em `em_falta` ou `reclamado` (acção prioritária, clicável)

Acessível pela navegação principal — não via CTA do Reconciliar.

---

## 6. Mapa dos Gaps no Código

| # | Gap | Função afectada | Linhas | Gravidade |
|---|---|---|---|---|
| 1 | Terceira via `Registado → Pago` não funciona | `processCUFText` | 1737, 1773 | Crítico |
| 2 | Clock dos 3 meses é event-based, não time-based | `applyCUF` sweep | 1938-1944 | Alto |
| 3 | `markAllReclaimed` sem sequência obrigatória | `markAllReclaimed`, `renderReclamar` | 1969-1998 | Alto |
| 4 | Painel de Recuperação não existe | — (criar novo) | — | Alto |
| 5 | `changeStatus` aceita qualquer transição | `changeStatus` | 1636-1644 | Médio |
| 6 | Estado `rejeitado` não existe | `STATUS_META`, UI detalhe | 1203-1208, 1606-1634 | Médio |
| 7 | Linhas da fatura sem match perdidas | `processCUFText` | 1790-1799 | Baixo |
| 8 | Edição manual de estado sem regras | — (não implementada) | — | Baixo |
| 9 | Reconciliação sobrescreve campo procedimento com lixo | `applyCUF` | 1919-1920, 1931-1932 | Alto |
| 10 | Match falha quando data fatura ≠ data acto (ex: admissão vs cirurgia) | `processCUFText`, `applyCUF` | 1770-1799 | Médio |

---

## 7. Priorização

### MVP — antes dos testes reais (Junho/Julho 2026)

1. **Gap 9** — procedimento preservado (15 min, protege dados) ✅ feito
2. **Gap 1** — bug terceira via (1-2h) ✅ feito (f7f975d)
3. **Gap 7** — contador de linhas sem match (30 min) ✅ feito (8cf2ace)
4. **Gap 2** — clock temporal autónomo (1-2h) ✅ feito (c78aac9)
5. **Gap 3** — sequência reclamação (2-3h) ✅ feito (9953707)
6. **Gap 10** — tolerância de datas ±3 dias (2-3h) ✅ feito
7. **Gap 4** — Painel de Recuperação real (4-6h) ✅ feito (dfea837)
8. **Gap 6** — estado Rejeitado (3-4h) ✅ feito (eaa8033)

### Pós-MVP

7. **Gap 5** — `canTransition` validado
8. **Gap 8** — UI de edição manual com regras

---

## 8. Prompts prontos para Claude Code

Cada prompt é self-contained. Abre Claude Code na pasta do projecto, cola o prompt.

### Prompt Gap 1 — Bug da terceira via

```
Lê @REFACTOR-ESTADOS.md, secções 4 e 6.

Tarefa: implementar a terceira via Registado → Pago via reconciliação.

Contexto: em index.html, função processCUFText (linha ~1736), a constante
ELIGIBLE inclui apenas 'em_falta' e 'reclamado'. Isto faz com que actos em
estado 'registado' que aparecem na fatura CUF sejam silenciosamente ignorados,
em vez de passarem a 'pago'.

Alterações:
1. Adicionar 'registado' ao set ELIGIBLE
2. Confirmar que applyCUF (linha ~1910) não tem lógica que dependa de o
   estado prévio ser em_falta/reclamado
3. Verificar o sweep de cutoff (linha ~1938-1944) — continua correcto porque
   um acto registado que deu match já foi marcado pago antes de lá chegar,
   e o filtro `p.status !== 'registado'` exclui-o

Teste mental: acto registado há 1 mês aparece na fatura → passa a pago. Sim.

Regras: só index.html. Commit: "Fix: reconciliação CUF passa actos registados directamente a pago".
```

### Prompt Gap 2 — Clock temporal

```
Lê @REFACTOR-ESTADOS.md, secção 4 (subsecção "Clock temporal").

Tarefa: criar clock temporal autónomo para marcar actos como em_falta.

Contexto: a transição registado → em_falta só acontece durante applyCUF
(linha ~1938). Se o utilizador não carregar fatura durante meses, actos
antigos nunca são marcados.

Alterações:
1. Criar função checkEmFaltaByTime() que percorre procedimentos e, para cada
   acto em 'registado' com data < hoje - 90 dias, marca como 'em_falta'.
2. Retornar o número de actos alterados.
3. Chamar checkEmFaltaByTime() no final do boot (após carregar dados do Drive)
   e em switchTab quando o utilizador volta ao dashboard ou à tab Reclamar.
4. Se algum acto foi alterado, chamar autoSave() e renderDashboard().
5. Depois deste clock estar em produção, o sweep dentro de applyCUF
   (linha ~1938-1944) torna-se redundante. Podes removê-lo ou manter por
   robustez — sugiro manter, não faz mal.

Regras: só index.html. Commit: "Feature: clock temporal marca actos em falta automaticamente".
```

### Prompt Gap 3 — Sequência da reclamação

```
Lê @REFACTOR-ESTADOS.md, secção 4 (subsecção "Marcar como Reclamado").

Tarefa: garantir que markAllReclaimed só corre depois de o utilizador ter
copiado o texto do email.

Contexto: renderReclamar (linha ~1969) mostra o texto do email e um botão
"Marcar todos como reclamados" sempre activo. O utilizador pode clicar antes
de copiar/enviar, ficando com actos marcados Reclamado sem terem sido
efectivamente reclamados.

Alterações:
1. Flag local window._relatorioCopiado = false ao entrar no separador.
2. Botão "Marcar todos como reclamados" começa disabled com opacity 0.5.
3. Em copyReclamar (linha ~1987), após copiar com sucesso, setar a flag
   e activar o botão.
4. Renomear o botão: "Confirmar envio e marcar como reclamado".
5. Info-box acima do botão: "Copia primeiro o email e envia à CUF. Só depois
   de enviares é que deves marcar como reclamado."

Regras: só index.html. Commit: "Fix: marcar como reclamado só após copiar relatório".
```

### Prompt Gap 7 — Contador de linhas sem match

```
Lê @REFACTOR-ESTADOS.md, secção 4 (subsecção "Reconciliação", ponto 6).

Tarefa: contar linhas da fatura CUF que não têm match e mostrar nota discreta.

Contexto: em processCUFText (linha ~1790), o loop sobre pdfByKey salta quando
!records.length. Não regista estas ocorrências. Perdemos sinal útil.

Alterações:
1. Contador linhasSemMatch que incrementa quando !records.length.
2. No HTML de output (linha ~1819), adicionar linha discreta em cinzento:
   "N linhas da fatura sem correspondência — ignoradas".
3. Guardar linhasSemMatch em window._pendingCUF para referência futura.

Sem UX nova, só informação.

Regras: só index.html. Commit: "Feature: contador de linhas sem correspondência na reconciliação".
```

### Prompt Gap 4 — Painel de Recuperação

```
Lê @REFACTOR-ESTADOS.md, secções 1, 3 e 5.

Tarefa: criar Painel de Recuperação como novo separador principal, distinto
do actual "Relatório" (que é casuística).

Contexto: renderRelatorio (linha ~2114) mostra estatísticas por mês/tipo/
especialidade — é casuística, não recuperação. O utilizador não tem ecrã
que mostre o estado da operação em euros.

Alterações:
1. Renomear o separador/função actual "Relatório" para "Casuística":
   - tab id mtab-casuistica
   - função renderCasuistica
   - preservar toda a lógica actual
2. Criar novo separador "Recuperação":
   - tab id mtab-recuperacao
   - função renderRecuperacao
3. Conteúdo do novo painel (usar estilo rel-card e info-box existentes):
   - Cartão "Total em risco": € em em_falta + € em reclamado (big number)
   - Cartão "Em reclamação activa": € em reclamado
   - Cartão "Total recuperado": € em pago. Sub-valores: este ano + acumulado.
   - Lista 10 casos mais antigos em em_falta ou reclamado, ordem crescente
     de data, clicáveis (abrem detalhe do acto).

Regras: só index.html. Commit: "Feature: Painel de Recuperação separado da Casuística".
```

### Prompt Gap 6 — Estado Rejeitado

```
Lê @REFACTOR-ESTADOS.md, secções 1, 2 e 4 (subsecção "Rejeitar").

Tarefa: adicionar estado terminal rejeitado para reclamações recusadas.

Alterações:
1. STATUS_META (linha ~1203): adicionar
   rejeitado: { label: 'Rejeitado', cls: 's-rejeitado' }
2. CSS: estilos .s-rejeitado e .ds-rejeitado em tom cinzento/discreto
   (não vermelho alarmante).
3. showDetail (linha ~1602): se p.status === 'reclamado', botão
   "CUF rejeitou reclamação". Abre prompt com select de motivo:
   "Código incorrecto" | "Doente não CUF" | "Fora de prazo" | "Outro"
   (se "Outro", pedir texto livre).
4. Função markRejected(id, motivo):
   p.status = 'rejeitado'
   p.rejeitadoEm = new Date().toISOString()
   p.motivoRejeicao = motivo
   autoSave + renderDashboard + toast
5. No Painel de Recuperação (gap 4): excluir rejeitados do "Total em risco"
   e "Em reclamação activa". Secção separada discreta:
   "Rejeitados: N casos · €X".
6. No detalhe de acto rejeitado, mostrar motivo e botão "Reverter rejeição"
   (volta a reclamado, limpa rejeitadoEm e motivoRejeicao).

Regras: só index.html. Commit: "Feature: estado Rejeitado para reclamações recusadas pela CUF".
```

### Prompt Gaps 9 + 10 — Proteger procedimento e tolerância de datas ✅ feito

```
Lê @REFACTOR-ESTADOS.md, secções 4 e 6.

Tarefa dupla:

A) GAP 9 — PROCEDIMENTO PRESERVADO
Em applyCUF, nos dois blocos que aplicam matches (clearMatches e resolved),
remover a linha:
  if (line.name) p.procedimento = line.name;
Manter a linha do valor (if (line.value != null) p.valor = line.value;).
Não remover a extracção de line.name em processCUFText — continua usada
na preview ao utilizador.

B) GAP 10 — TOLERÂNCIA DE DATAS
Após a lógica de matching estrito em processCUFText, adicionar fase 2:
para cada linha CUF sem match, procurar actos com mesmo normProc(processo),
estado em {registado, em_falta, reclamado}, não emparelhados, e diferença
absoluta de datas ≤ 3 dias. Se exactamente 1 candidato → toleranceMatch.
Zero ou múltiplos → ignorar (linha sem match).
Na UI: bloco amarelo com checkboxes por par (default: não marcado),
listando doente, data registada, data fatura, diferença em dias, valor.
Botão "Aplicar seleccionados" separado do botão principal.
Nunca aplicar automaticamente.

Regras: só index.html. Commit: "Fix: preservar procedimento e adicionar tolerância de datas na reconciliação".
```

---

## 9. Questões em aberto (pensar depois dos testes reais)

- **Re-reclamações**: se a CUF ignora primeira reclamação em muitos casos, vai ser preciso `contador_reclamacoes` ou `ultima_reclamacao_em`. Decisão actual: overkill no MVP.
- **Limbo eterno em Reclamado**: se a CUF nunca responde, um acto fica `reclamado` para sempre. Auto-rejeitar após X meses? Pós-MVP.
- **Bootstrap de casuística histórica**: import XLSX para criar actos directamente em `pago`. Pós-MVP, operação one-off.
- **Grupo 3 auto-populate**: "CUF pagou-me algo que não tinha registado" — decisão depois dos primeiros testes reais.

---

---

## 10. Estado de implementação (Abril 2026)

Todos os gaps MVP estão implementados. Ver commits em `CLAUDE.md`.

**Camada UI (backend inalterado):**
Os 5 estados internos continuam exactamente como definidos. A UI expõe 3 categorias visuais via `getUIBucket(status)`:
- `registado` → "Recente" (dot cinza neutro)
- `em_falta` / `reclamado` → "Em dívida" (dot vermelho; reclamado mostra tag "enviado em MM/AA")
- `pago` → "Recebido" (dot verde)
- `rejeitado` → visível com estilo cinza existente (label "Rejeitado")

Separador "Reclamar" removido da navegação. Fluxo de reclamação integrado no separador "Cruzar dados" via `renderDebtBlock()`.

**Pendente pós-MVP:**
- Gap 5 — `canTransition` validado
- Gap 8 — UI de edição manual de estado com regras
- Demo mode — caso de ambiguidade (2 actos mesmo doente+data)
