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
- Reconciliação com listagem CUF (upload/paste PDF ou texto)
- Status flow: Registado → Em Falta → Reclamado → Pago
- Regra dos 3 meses para marcar "Em Falta"
- Painel de Recuperação
- Relatório de reclamação (exporta para `.txt` e clipboard)
- Dados separados por utilizador
- Botão "Sair da aplicação" (ecrã de logout + limpeza `?dev`)
- Demo mode funcional (incompleto — falta caso de ambiguidade e cruzamento gastro)
- Calendário com espaçamentos corrigidos
- Google login funciona à primeira tentativa (botão desabilitado até SDK carregar)
- Account picker Google sempre visível (prompt: 'select_account')
- Sem flash de ecrã de login após autenticação
- Desktop: calendário e lista de actos do dia lado a lado

### Não funciona / falta
- **Auth real** — falta whitelist/controlo de acesso e publicação OAuth
- **Demo mode incompleto** — falta caso de ambiguidade (2 actos mesmo doente+data) e caso gastro no cruzamento

---

## BUGS ACTIVOS

| # | Bug | Notas |
|---|-----|-------|
| 1 | Gemini 429 rate limit errors (frequente desde cortes no free tier, Abril 2026) | — |

---

## MELHORIAS PENDENTES

| # | Melhoria | Prioridade |
|---|----------|------------|
| 1 | Demo mode — adicionar caso de ambiguidade (2 actos mesmo doente+data) e caso gastro no cruzamento | Alta |
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
