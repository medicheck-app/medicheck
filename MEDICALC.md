# MediCalc — Contexto para Claude Code

> Calculadora de doses pediátricas para anestesia. Projecto irmão do Medicheck.
> Ficheiro lido manualmente quando se trabalha no MediCalc (não é carregado automaticamente — o CLAUDE.md desta pasta é do Medicheck).

---

## PROJECTO

Calculadora mobile-first de doses de fármacos pediátricos e sinais vitais por faixa etária, para uso em bloco operatório.

- **Localização:** `C:\Users\treta\OneDrive\Área de Trabalho\MediCalc\index.html`
- **GitHub:** https://github.com/medicheck-app/MediCalc
- **Deploy:** a definir (GitHub Pages como o Medicheck é a opção natural — activar em Settings → Pages → branch main)

---

## STACK TÉCNICA

- **Ficheiro único:** `index.html` (~1462 linhas, React 18 transpilado inline + CSS + dados)
- **React via CDN** — sem build step, sem JSX puro (código já transpilado com Babel)
- **Sem dependências locais** — abre directamente no browser
- **Logótipo:** `Gemini_Generated_Image_2mx0vu2mx0vu2mx0.png` (1.8 MB, na mesma pasta)

---

## ESTRUTURA DO CÓDIGO

| Secção | Linhas (aprox.) | Descrição |
|--------|----------------|-----------|
| Constants | 35–42 | Cores: AMBER, BG, CARD, BORDER, TEXT, MUTED, DIM, URGENT |
| `drugs[]` | 45–576 | Array com todas as categorias e fármacos |
| `vitalSigns[]` | 577–637 | Sinais vitais por faixa etária |
| `fmt()` | 638–641 | Arredondamento a 2 casas decimais |
| `StridorPrepBox` | 643–823 | Componente especial para Estridor/Croup |
| `DoseRow` | ~824 | Linha standard de fármaco |
| `DualRouteRow` | ~860 | Fármaco com 2 vias (ex: inalado + aerossol) |
| `VitalRow` | ~990 | Linha de sinais vitais |
| `MediCalc` | 1111–fim | Componente principal (5 useState hooks) |

---

## FÁRMACOS COBERTOS

| Categoria | Fármacos |
|-----------|----------|
| Indução | Propofol, Fentanil |
| Relaxante Muscular | Succinilcolina, Rocurónio, Sugammadex |
| Analgesia | Paracetamol, Tramadol, Metamizol, Cetorolac |
| Anestesia Local | Ropivacaína, Lidocaína |
| Antieméticos | Ondansetron, Droperidol, Dexametasona |
| Broncodilatadores | Salbutamol, Ipratrópio, Beclometasona |
| Estridor/Croup | L-Adrenalina, Dexametasona PO, Budesonido |
| Emergência | Adrenalina, Atropina, Naloxona, Flumazenil, Clemastina, Hidrocortisona, Glicose 10% |

---

## FUNCIONALIDADES

- **Inputs:** peso (kg) + idade (anos, aceita decimais — ex: 0.5 para 6 meses)
- **Separador Doses:** dose calculada (mg) + volume (ml) para cada fármaco, com máximos
- **Separador Sinais Vitais:** FC, FR, TAS, TAD, SpO2, Glicemia por faixa etária — faixa correspondente à idade introduzida fica destacada a âmbar
- **Tubo endotraqueal:** Cole formula — cuffed = idade/4 + 3.5 | uncuffed = idade/4 + 4 (só ≥ 2 anos)
- **StridorPrepBox:** calcula L-Adrenalina neb (0.5 ml/kg, máx 5 ml), SF para completar 4 ml mínimo, e indica se pode misturar Budesonido (volume ≤ 3 ml)

---

## REGRAS DE CÓDIGO

- **Só modificar `index.html`** — não há outros ficheiros de código
- **Não usar JSX** — o código está transpilado; usar `React.createElement(...)` directamente
- **Fórmulas médicas** verificadas — não alterar sem confirmar com fontes pediátricas
- **Commits em português** (convenção do projecto)
- **`_useState0` na linha 1131** — naming confuso (devia ser `_useState10`) mas não é bug funcional; não tocar

---

## HISTÓRICO DE BUGS CORRIGIDOS

| Data | Linha | Bug | Fix |
|------|-------|-----|-----|
| 2026-04-28 | 580 | `ageMax: 0` — faixa "< 1 ano" nunca destacava (só funcionava para idade = 0 exactamente) | `ageMax: 0.99` |
| 2026-04-28 | 246 | Nome fármaco `"Ondansetrom"` (errado) | `"Ondansetron"` |

---

## SINAIS VITAIS — FAIXAS ETÁRIAS

| Faixa | ageMin | ageMax | FC | FR |
|-------|--------|--------|----|----|
| < 1 ano | 0 | 0.99 | 100–160 | 30–60 |
| 1–2 anos | 1 | 2 | 90–150 | 24–40 |
| 3–5 anos | 3 | 5 | 80–140 | 22–34 |
| 6–9 anos | 6 | 9 | 70–120 | 18–30 |
| 10–11 anos | 10 | 11 | 60–110 | 16–22 |
| 12–18 anos | 12 | 18 | 60–100 | 12–20 |
