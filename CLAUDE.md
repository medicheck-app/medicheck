# MediCheck — Instruções para Claude Code

## Contexto
MediCheck é uma PWA para médicos do CUF Torres Vedras registarem actos médicos.
Suporta 3 especialidades: Anestesiologia, Cirurgia (9 sub-especialidades) e Gastroenterologia.
Fluxo: fotografar vinheta → OCR → registar acto → reconciliar com honorários CUF → relatório de reclamação.

## Regras obrigatórias
- Todas as alterações devem ser feitas APENAS em `index.html` (é um single-file app)
- NUNCA tocar em `dev.html`, `apikey.txt`, ficheiros `.jpg`, `.pdf` ou `privacy.html`
- Manter o estilo de código existente: CSS custom properties (--bg, --ink, --accent, etc.), sem frameworks
- Respeitar as 3 especialidades e os seus temas de cor (amber/azul/verde)
- O HTML deve ser válido — verificar antes de fazer commit
- Mensagens de commit em português
- Não quebrar a encriptação Google Drive (AES-256-GCM via PBKDF2)

## Arquitectura
- Single file HTML/CSS/JS (~3500 linhas)
- CSS custom properties para theming (dark theme, specialty colors)
- OCR via Cloudflare Worker proxy → Claude Vision API
- Dados encriptados no Google Drive do utilizador (AES-256-GCM, PIN via PBKDF2)
- localStorage como fallback/cache
- Deploy automático via GitHub Pages (branch main)
- Libs externas: XLSX.js, Google OAuth 2.0 (loaded via CDN)

## Status dos actos
registado → em_falta (≥3 meses) → reclamado → pago / arquivado

## Ao criar PRs
- Título curto em português
- Descrever o que mudou e porquê no corpo do PR
- Se tocou em CSS, mencionar se afecta layout mobile
- Se tocou em JS de dados/sync, confirmar que não quebra a encriptação
