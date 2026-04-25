# Medicheck — Painel de Navegação

> Abre este ficheiro no Obsidian para navegar rapidamente para qualquer documento do projecto.

---

## Contexto para Claude Code

| Ficheiro | Para que serve |
|----------|---------------|
| [[CLAUDE]] | Contexto principal — carregado automaticamente em cada sessão |
| [[REFACTOR-ESTADOS]] | Estado actual da máquina de estados — actualizar quando a lógica mudar |
| [[ESTADOS]] | Documento de design original completo — referência para decisões de arquitectura |
| [[CHANGELOG]] | Log de sessões — historial de o que mudou e quando |

---

## Estado do projecto (Abril 2026)

- **4 separadores:** Registar · Cruzar dados · Pagamentos · Relatório
- **5 estados internos:** `registado` → `em_falta` → `reclamado` → `pago` | `rejeitado`
- **Todos os gaps MVP concluídos** — ver [[REFACTOR-ESTADOS]] para detalhes
- **Parser PDF validado** com 3 listagens reais da CUF (sessão 2026-04-25, fix crítico: 0→270 matches). _Matching_ propriamente dito ainda por validar (PDFs históricos têm NPs antigos, BD tem NPs novos — convergem em Jun/Jul)
- **Risco activo:** matching BD↔fatura por testar com dados reais (esperado Jun/Jul 2026); 4 melhorias implementadas (0c45a15): nome BD vs CUF, tolerância ±7d, fuzzy Phase 3, auditoria matchMethod
- **Save Drive:** fiabilidade reforçada (commit 79e473e, Abril 2026)
- **Calendário:** swipe lateral com animação (commit 9f478a1, Abril 2026)

---

## Ficheiro principal

```
index.html   ← toda a lógica da app (única fonte de verdade para código)
dev.html     ← cópia local para testes (nunca publicar)
```

---

## Regras rápidas

- Só modificar `index.html`
- Commits em português
- Bugs CSS mobile: ler o grid antes de adivinhar valores
- Sync/persistência: nunca overwrite — sempre merge
