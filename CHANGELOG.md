# Medicheck — Log de Sessões

> Uma linha por sessão. Actualizado automaticamente pelo Claude Code no fim de cada sessão.

---

| Data | Tipo | Resumo |
|------|------|--------|
| 2026-04-28 | Design | Painel de Desempate: layout 3 colunas + comportamento Confirmar/Ignorar/fila linear definidos; implementação pendente |
| 2026-04-28 | Feature | Parser PDF extrai campo `procedimento` (Acto Médico) via coordenadas X/Y; threshold 52% largura página, janela ±18/35px; campo adicionado a cada `cufLine` sem tocar em matching |
| 2026-04-28 | Fix | 4 bugs corrigidos: `var(--line)` indefinida→`var(--border)` (2 ocorrências), `applyToleranceCUF()` sem auditoria matchMethod, `renderReclamar()` código morto removida; `.gitignore` protege ficheiros sensíveis (recovery codes, PDFs, CSV) |
| 2026-04-28 | Fix (MediCalc) | 2 bugs corrigidos: ageMax 0→0.99 (faixa "< 1 ano" nunca destacava) e ortografia Ondansetron; git init + primeiro commit; MEDICALC.md criado no vault |
| 2026-04-25 | Feature | Motor matching: 4 melhorias pré-teste real — confirmação de nome BD vs CUF (Jaccard), tolerância ±7d com UI 1:N, Phase 3 fuzzy nome+30d, auditoria matchMethod gravada em cada procedimento |
| 2026-04-25 | Fix crítico | Parser PDF reescrito (data e NP independentes, NP 5-10 dígitos) — antes 0 matches em 1.569 linhas reais, agora 270 actos recuperados nos 3 PDFs históricos; criado `_dump_pdf.html` para validação offline |
| 2026-04-24 | Feature + Fix | Swipe lateral no calendário com animação translateX; 3 bugs corrigidos (transform não resetado, animating não resetado, sem listener touchmove) |
| 2026-04-24 | Fix | Fiabilidade save Drive: await pós-login, response.ok PATCH/POST, beforeunload guard, toast erro universal |
| 2026-04-20 | Fix | 4 bugs extracção PDF CUF: regex data DD-MM-YY, estornos ignorados, Y-grouping ±5px, nomes só palavras CAPS |
| 2026-04-20 | Documentação | Sincronização completa: criado REFACTOR-ESTADOS.md e _HOME.md; CLAUDE.md actualizado (anestesia sem UI, rename no backlog); memória consolidada de 8 para 4 ficheiros; wiki-links adicionados; CHANGELOG criado |
| 2026-04-19 | Feature | Refactor UI: checkbox para incluir actos já reclamados no email; bloco em dívida só aparece após cruzamento; total recuperado removido do resumo pós-cruzamento |
| 2026-04-18 | Fix + Feature | Gap 1 (3ª via registado→pago); PIN touch-action 300ms; botão logout com ecrã dedicado |
| 2026-04-17 | Fix | Google login à primeira tentativa; account picker sempre visível; sem flash após auth; desktop layout |
| 2026-04-14 | Fix | Gastro no relatório; procedimento personalizado cirurgia não desaparecia |
| 2026-04-13 | Fix | Política privacidade inline; scroll lista mobile; OCR só dígitos; OCR seguradora |
