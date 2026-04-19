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

---

## REGRAS INVIOLÁVEIS

- Só modificar `index.html` — **nunca** tocar em `dev.html`, `apikey.txt`, ficheiros `.jpg`, `.pdf`, ou `privacy.html`
- Commits Git escritos em **português**
- Não fazer perguntas de diagnóstico — implementar com base no contexto disponível

---

## ESTADO ACTUAL

### Funciona
- Onboarding completo (Google Sign-in → PIN → Especialidade)
- Registo de actos
- OCR via foto (anestesia) — extrai nº processo (só dígitos) e seguradora
- Reconciliação com listagem CUF (upload/paste PDF ou texto) — inclui 3ª via: actos `registado` que aparecem na fatura passam directamente a `pago` (fix Gap 1, commit f7f975d)
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
- Reconciliação: tolerâncias ±3 dias aplicadas automaticamente; aviso discreto no preview
- Reconciliação: após aplicar, mostra "X doentes em falta" a vermelho se houver em_falta não resolvidos
- Reconciliação: `totalRecuperado` conta só `reclamado→pago` (não registado→pago)
- Pagamentos: mostra só `reclamado` pendentes (pagos removidos da lista após confirmação)
- Demo mode: exercita tolerância ±3 dias (FERNANDO SANTOS PEREIRA, diff=2 dias); linha de consulta na fatura simulada; todos os registos com `valor` preenchido
- Calendário com espaçamentos corrigidos
- Google login funciona à primeira tentativa (botão desabilitado até SDK carregar)
- Account picker Google sempre visível (prompt: 'select_account')
- Sem flash de ecrã de login após autenticação
- Desktop: calendário e lista de actos do dia lado a lado
- Teclado PIN sem delay de 300ms em mobile (`touch-action: manipulation`)

### Não funciona / falta
- **Auth real** — falta whitelist/controlo de acesso e publicação OAuth
- **Demo mode** — falta caso de ambiguidade (2 actos mesmo doente+data)

---

## BUGS ACTIVOS

| # | Bug | Notas |
|---|-----|-------|
| 1 | Gemini 429 rate limit errors (frequente desde cortes no free tier, Abril 2026) | — |

---

## MELHORIAS PENDENTES

### Refactor de estados (ESTADOS.md — ver prompts prontos no ficheiro)

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

---

## RISCO CRÍTICO ACTIVO

**Reconciliação nunca testada com dados reais.** Primeiro teste real esperado Junho/Julho 2026. É o item de risco mais crítico do MVP — tudo o resto é secundário até isto estar validado.
