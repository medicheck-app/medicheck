# MEDICHECK — CONTEXTO PARA CLAUDE CODE

> Este ficheiro é lido automaticamente pelo Claude Code antes de cada sessão.
> Mantém-no actualizado. A fonte de verdade para estratégia e produto é a conversa principal com o Claude (claude.ai).

---

## PROJECTO

PWA mobile-first para médicos registarem actos operatórios e reconciliarem com listagens de honorários da CUF — recuperando pagamentos em falta.

- **Produção:** https://medicheck-app.github.io/medicheck/
- **Google Cloud Project:** medicheck-489414
- **Limite actual:** 100 utilizadores de teste (OAuth não publicado)

---

## STACK TÉCNICA

- **Ficheiro principal:** `index.html` (~3300 linhas, vanilla JS + HTML + CSS inline)
- `dev.html` é cópia local para testes — nunca é publicada directamente
- **OCR:** Gemini Flash via Cloudflare Worker (chave API segura, não exposta no cliente)
- **Auth:** Google Sign-in → PIN → encriptação AES-256-GCM com chave derivada via PBKDF2
- **Storage:** Google Drive (appDataFolder) para dados encriptados; localStorage como cache (sincronização entre dispositivos não confirmada — risco activo)
- **Importação:** XLSX.js para mapas CUF + paste de PDF/texto
- **Deploy:** editar `dev.html` → copiar para `index.html` → `git push` → GitHub Pages
- **Notas:** `CLAUDE.md` editável no Obsidian (vault = pasta `Medicheck`). Editar entre sessões para manter contexto actualizado.
- **Docs no vault:** `CLAUDE.md` (contexto sessão), `REFACTOR-ESTADOS.md` (estado da máquina de estados), `ESTADOS.md` (design original), `_HOME.md` (painel de navegação Obsidian)

---

## REGRAS INVIOLÁVEIS

- Só modificar `index.html` — **nunca** tocar em `dev.html`, `apikey.txt`, ficheiros `.jpg`, `.pdf`, ou `privacy.html`
- Commits Git escritos em **português**
- Não fazer perguntas de diagnóstico — implementar com base no contexto disponível

---

## WORKFLOW CONVENTIONS

- **Após qualquer tarefa concluída**, executar automaticamente sem esperar instrução:
  1. Preview para verificar (se aplicável)
  2. `git add index.html && git commit` com mensagem convencional em português
  3. `git push`
  4. Actualizar `CLAUDE.md` se algo mudou no estado/funcionalidades/stack
  5. Actualizar [[REFACTOR-ESTADOS]] se lógica de estados mudou
  6. Actualizar `_HOME.md` (secção "Estado do projecto") se o estado geral mudou
  7. Acrescentar linha ao `CHANGELOG.md` (data + tipo + resumo 1 linha)
  8. Gerar handoff prompt pronto a colar na próxima sessão
- Excepção: se o utilizador disser "só rascunho" ou "draft only", não commitar.
- Em sessões apenas de documentação (sem alterações ao `index.html`): saltar passos 1-3, executar 4-7.

---

## ARQUITECTURA DO PROJECTO

- **Single-file PWA** — toda a lógica está em `index.html`, sem build step
- **Sistema de vistas:** usa classes `.screen` + `.active` para mostrar/esconder ecrãs — **nunca usar `display:` inline** que conflitue com este sistema
- **Ao adicionar um novo ecrã:** actualizar SEMPRE o HTML/CSS **e** o handler JS correspondente (ex: `showScreen('nome')`, `handleLogout`) — omitir o handler faz o ecrã nunca aparecer
- **Ao adicionar botões/acções:** verificar se existe um event listener ou se é preciso criar um novo
- **Preview:** usar `mcp__Claude_Preview__preview_eval` — não arrancar servidores manuais

---

## REGRAS DE DEBUGGING

- **Antes de editar qualquer bug:** reformular o bug em palavras próprias + descrever comportamento esperado vs actual numa frase. Se houver ambiguidade, fazer UMA pergunta de esclarecimento.
- **Bugs de sync/persistência:** NUNCA sobrescrever dados locais com remotos — sempre fazer merge (remote → local) para evitar perda de dados (incidente anterior: pacientes apagados por overwrite do Drive)
- **Bugs CSS mobile:** antes de tentar corrigir, inspecionar o template grid (`minmax`, `fr`, `auto`) e `touch-action` — não adivinhar valores sem ler o CSS existente
- **Bugs de layout que falham 2+ vezes:** entrar em Plan mode, listar hipótese + linhas exactas a tocar + riscos, aguardar aprovação antes de editar

---

## ESTADO ACTUAL

### Funciona
- Onboarding completo (Google Sign-in → PIN → Especialidade)
- Registo de actos
- OCR via foto (anestesia) — extrai nº processo (só dígitos) e seguradora
- Reconciliação com listagem CUF (upload/paste PDF ou texto) — inclui 3ª via: actos `registado` que aparecem na fatura passam directamente a `pago` (fix Gap 1, commit f7f975d)
- Parser PDF suporta formato real CUF: datas `DD-MM-YY` (traços + ano 2 dígitos), estornos negativos ignorados (não marcam actos como pagos), nomes extraídos só de palavras em CAPS, tolerância Y±5px para células com texto wrappado (commit ee63a7a)
- Parser PDF: data e NP procurados independentemente (ordem real CUF é `DATA NP NOME`, não `NP ... DATA` como estava); NP aceita 5-10 dígitos (cobre transição CUF). Validado contra 3 PDFs reais (270 actos extraídos correctamente, 0 antes do fix) — sessão 2026-04-25
- Status flow: Registado → Em Falta → Reclamado → Pago
- Regra dos 3 meses para marcar "Em Falta" — clock temporal autónomo (`checkEmFaltaByTime()`) corre ao entrar com PIN e ao abrir separador "Cruzar dados"
- Estado `rejeitado` — actos reclamados recusados pela CUF, com motivo, reversíveis
- UI de 3 categorias visuais: "Registado" (registado), "Em dívida" (em_falta+reclamado), "Recebido" (pago); rejeitados visíveis com estilo cinza
- Navegação: Registar · Cruzar dados · Pagamentos · Relatório (4 separadores)
- Separador "Relatório": cartões de faturação (em dívida, em reclamação, total recuperado) + casuística por mês/tipo/especialidade + exportar Excel
- Fluxo de reclamação integrado no separador "Cruzar dados": bloco "Actos em dívida" aparece só após cruzamento; checkbox para incluir actos já reclamados no email; botão confirmar só activo após copiar
- Dados separados por utilizador
- Botão "Sair da aplicação" (ecrã de logout + limpeza `?dev`)
- Reconciliação: ignora linhas "consulta" da fatura (não aplicável a actos operatórios)
- Reconciliação: tolerâncias ±7 dias (1:1 automático; 1:N → UI manual toleranceAmbiguous)
- Reconciliação: Phase 3 fuzzy nome + janela 30 dias (sempre manual, com barra de confiança visual)
- Reconciliação: confirmação de nome BD vs CUF no preview — alerta amber se similaridade Jaccard < 50%
- Reconciliação: auditoria `matchMethod` (`exact`/`tolerance`/`fuzzy_name`) + `matchDaysDiff`/`matchSimilarity` gravados em cada procedimento
- Reconciliação: após aplicar, mostra "X doentes em falta" a vermelho se houver em_falta não resolvidos
- Reconciliação: `totalRecuperado` conta só `reclamado→pago` (não registado→pago)
- Pagamentos: mostra só `reclamado` pendentes (pagos removidos da lista após confirmação)
- Demo mode: exercita tolerância (FERNANDO SANTOS PEREIRA, diff=2 dias, dentro do novo limite ±7d); linha de consulta na fatura simulada; todos os registos com `valor` preenchido
- Calendário com espaçamentos corrigidos
- Swipe lateral no calendário: dedo segue com `translateX` durante drag; no `touchend` mês actual anima para fora e novo mês entra do lado oposto (`transition: transform 300ms`); `touch-action: pan-y` evita conflito com scroll vertical
- Google login funciona à primeira tentativa (botão desabilitado até SDK carregar)
- Account picker Google sempre visível (prompt: 'select_account')
- Sem flash de ecrã de login após autenticação
- Desktop: calendário e lista de actos do dia lado a lado
- Teclado PIN sem delay de 300ms em mobile (`touch-action: manipulation`)
- Save para Drive com fiabilidade reforçada: `await` no sync pós-login, `response.ok` em PATCH/POST, `beforeunload` guard durante save activo, toast de erro em qualquer falha (commit 79e473e)

### Não funciona / falta
- **Auth real** — falta whitelist/controlo de acesso e publicação OAuth
- **Demo mode** — falta caso de ambiguidade (2 actos mesmo doente+data)
- **Tipo `anestesia` sem UI** — existe no modelo de dados e nos relatórios, mas não há botão na UI para criar actos de anestesia manualmente; actos deste tipo só surgem via reconciliação CUF

---

## BUGS ACTIVOS

| # | Bug | Notas |
|---|-----|-------|
| 1 | Gemini 429 rate limit errors (frequente desde cortes no free tier, Abril 2026) | — |

---

## MELHORIAS PENDENTES

### Refactor de estados ([[ESTADOS]] — ver prompts prontos no ficheiro)

| Gap | Descrição | Prioridade |
|-----|-----------|------------|
| ~~1~~ | ~~Terceira via Registado → Pago~~ | ~~Crítico~~ — ✅ feito (f7f975d) |
| ~~9~~ | ~~Reconciliação sobrescreve `procedimento` com lixo~~ | ~~Alto~~ — ✅ feito (0c7b2c3) |
| ~~10~~ | ~~Match falha quando data fatura ≠ data acto~~ | ~~Médio~~ — ✅ feito (0c7b2c3) |
| ~~7~~ | ~~Contador de linhas da fatura sem match~~ | ~~Baixo~~ — ✅ feito (8cf2ace) |
| ~~2~~ | ~~Clock temporal autónomo para marcar `em_falta` (`checkEmFaltaByTime()`)~~ | ~~Alto~~ — ✅ feito (c78aac9) |
| ~~3~~ | ~~Sequência obrigatória na reclamação (botão só activo após copiar email)~~ | ~~Alto~~ — ✅ feito (9953707) |
| ~~4~~ | ~~Painel de Recuperação real (separado da Casuística/Relatório)~~ | ~~Alto~~ — ✅ feito (dfea837) |
| ~~6~~ | ~~Estado `rejeitado` para reclamações recusadas~~ | ~~Médio~~ — ✅ feito (eaa8033) |

### Outros

| # | Melhoria | Prioridade |
|---|----------|------------|
| ~~1~~ | ~~Demo mode redesenhado com dados realistas~~ | ~~Alta~~ — ✅ feito (6d38dc0) |
| ~~2~~ | ~~Demo mode — tolerância ±3 dias e regra de consultas ignoradas~~ | ~~Média~~ — ✅ feito (esta sessão) |
| 1 | Demo mode — adicionar caso de ambiguidade (2 actos mesmo doente+data) | Média |
| 2 | IP local `192.168.1.186:8000` como authorized origin no GCP para testes iPhone | Baixa |

---

## DECISÕES DE PRODUTO TOMADAS

- Match por nº processo, não por nome (CUF não confirma linha a linha)
- Copy-paste é aceitável (CUF protege o Sheets contra download)
- `pagos[]` mantido apesar de redundante — reservado para analytics/monetização futura
- `panel-relatorio` só acessível via CTA do Reconciliar — aceite por design
- PDF reading: funcional mas limitado — apenas PDFs pesquisáveis, apenas extrai nº processo
- OCR: Gemini API via Cloudflare Worker (não Claude Haiku)

---

## ARQUITECTURA OCR

```
[Cliente] → foto → [Cloudflare Worker] → Gemini Flash API → resultado OCR → [Cliente]
```
A chave Gemini não é exposta no cliente. O Worker faz a chamada à API.

---

## BACKLOG PÓS-MVP

- Breakdown mensal do valor recuperado no Painel de Recuperação (total por mês + total acumulado)
- Mais especialidades (gastroenterologia, cirurgia, consulta)
- Auth real com whitelist + publicação OAuth
- Renomear o site (nome actual considerado pouco profissional para contexto CUF)

---

## RISCO CRÍTICO ACTIVO

**Matching BD↔fatura nunca testado com dados reais.** Parser foi validado contra 3 PDFs históricos da CUF (sessão 2026-04-25, 270 actos extraídos correctamente), mas o **matching** propriamente dito não pôde ser testado: os PDFs históricos têm NPs de 6 dígitos (sistema antigo), a BD tem 7-9 dígitos (sistema novo). Em Junho/Julho convergem — é aí que se valida o matching.

Melhorias implementadas (sessão 2026-04-25, commit 0c45a15):
1. ✅ Confirmação de nome: preview mostra BD vs CUF com alerta amber se Jaccard < 50%
2. ✅ Tolerância ±7 dias: 1:1 automático; 1:N → UI manual `toleranceAmbiguous` (não bloqueia Aplicar)
3. ✅ Phase 3 fuzzy nome + janela 30 dias: sempre manual, com barra de confiança visual
4. ✅ Auditoria: `matchMethod` (`exact`/`tolerance`/`fuzzy_name`) + `matchDaysDiff`/`matchSimilarity` gravados em cada procedimento
