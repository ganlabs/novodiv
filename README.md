# NovoDiv

> **Divisor automático de PDFs de Petição Inicial de processos jurídicos brasileiros.**

[![Status](https://img.shields.io/badge/status-MVP-yellow)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()
[![Offline](https://img.shields.io/badge/100%25-offline-blue)]()

NovoDiv recebe PDFs **integrais** de processos (PJe, Projudi, eProc, PROCON) e separa automaticamente a **Petição Inicial** do restante do processo, gerando dois arquivos:

- `Inicial_{CNJ}.pdf` — só a petição inicial
- `Docs_{CNJ}.pdf` — o resto do processo (antes + depois)

Os arquivos **nunca saem do seu computador** — tudo roda no navegador, sem backend, sem upload, sem OCR.

---

## ✨ Funcionalidades

- 📥 **Drag & drop** ou seleção de múltiplos PDFs integrais
- 🔍 **Detecção automática** do intervalo de páginas da Petição Inicial
- 🏛️ **Suporte multi-sistema:** PJe · Projudi · eProc · PROCON Estadual
- ✂️ **Split nativo** via `pdf-lib` (sem perda de qualidade)
- 📊 **Relatório de revisão** com score de confiança (0-100%) e flag `👁️ Revisar` automática
- 🖥️ **Sem instalação** — abre direto no navegador
- 🌐 **Funciona offline** (com fallback para CDN se primeira vez sem internet)

---

## 🚀 Como usar

1. Abra o arquivo **`index.html`** no Chrome ou Edge (navegadores com suporte à **File System Access API**)
2. Arraste os PDFs integrais para a área de upload
3. Clique em **"⚡ Processar e dividir"**
4. Escolha a pasta de destino (o navegador vai pedir permissão)
5. Revise o relatório ao final

> ⚠️ **Navegadores suportados:** Chrome, Edge, Opera. Firefox e Safari não suportam a File System Access API usada para salvar os PDFs.

---

## 📁 Estrutura do repositório

```
novodiv/
├── index.html              # 🟢 O APP INTEIRO (HTML + CSS + JS)
├── logo.png                # logo Gondim (topo)
├── favicon.png             # ícone do app
│
├── processos/              # 📥 ENTRADA: PDFs integrais a processar
├── NOVOS/                  # 📥 ENTRADA: PDFs novos a processar
├── erros/                  # 📥 ENTRADA: PDFs com problemas (gera "erros" no log)
├── procon/                 # 📥 ENTRADA: PDFs específicos de PROCON
│
├── teste1/                 # 📤 SAÍDA: PDFs já divididos (Inicial_* + Docs_*)
│
├── docs/
│   └── LEITURA_PROFUNDA.md # análise técnica do algoritmo
│
├── README.md               # este arquivo
├── SPEC.md                 # especificação técnica
├── AGENTS.md               # guia para agentes de IA
└── LICENSE                 # MIT
```

---

## 🧠 Como funciona (resumo)

O coração do sistema é a função `detectInicialRange()` em `index.html` (Algorithm v7). Ela opera em **3 fases**:

| Fase | Objetivo | Estratégia |
|------|----------|------------|
| **0** | Detectar PROCON | Procura marcadores `TERMO DE NOTIFICAÇÃO` / `PROCON ESTADUAL` nas primeiras 20 páginas |
| **1** | Encontrar **início** da petição | Regex de vocativos (`EXCELENTÍSSIMO`, `AO DOUTO JUÍZO`...) + refinamentos `INIC1` / `TERMO2` / `pattern_alt` |
| **2** | Encontrar **fim** da petição | 5 candidatos concorrentes: mudança de ID de doc, novo doc por conteúdo, fechamento + OAB, etc. Pega o mais conservador |

O resultado é um **score de confiança 0-5**, mapeado para porcentagem no relatório. Casos com score < 3 recebem flag `👁️ Revisar`.

📖 Análise técnica completa: [`docs/LEITURA_PROFUNDA.md`](docs/LEITURA_PROFUNDA.md)
📐 Especificação detalhada: [`SPEC.md`](SPEC.md)

---

## 🛠️ Stack técnica

- **Linguagem:** JavaScript ES2020+ (vanilla, sem build)
- **PDF read:** [pdf.js](https://mozilla.github.io/pdf.js/) v3.11.174 (via CDN)
- **PDF write:** [pdf-lib](https://pdf-lib.js.org/) v1.17.1 (via CDN)
- **Persistência local:** File System Access API (Chrome/Edge)
- **Sem dependências locais:** zero `npm install`, zero build step

---

## 🤝 Contribuição

1. Fork o projeto
2. Crie uma branch: `git checkout -b feat/minha-melhora`
3. Commit: `git commit -m "feat: ..."`
4. Push: `git push origin feat/minha-melhora`
5. Abra um Pull Request

Veja [`AGENTS.md`](AGENTS.md) para convenções de código, padrão de mensagens de commit e processo de revisão.

---

## 📜 Licença

MIT — veja [`LICENSE`](LICENSE).

---

## 🏢 GanLabs

NovoDiv é mantido por [GanLabs](https://github.com/ganlabs) — laboratório de ferramentas jurídicas com privacidade por padrão.
