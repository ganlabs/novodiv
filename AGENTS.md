# AGENTS.md — Guia para Agentes de IA

> Instruções, convenções e regras para agentes de IA que trabalham no **NovoDiv**.
> Este arquivo é lido **antes** de qualquer alteração no código.

---

## 1. O que é o NovoDiv

App single-file (`index.html`) escrito em **HTML + CSS + JavaScript vanilla** que divide PDFs integrais de processos jurídicos brasileiros, extraindo automaticamente a Petição Inicial. **Sem backend, sem build, sem dependências locais.**

Antes de sugerir mudanças, **leia**:
- [`README.md`](README.md) — visão geral e uso
- [`SPEC.md`](SPEC.md) — especificação técnica completa
- [`docs/LEITURA_PROFUNDA.md`](docs/LEITURA_PROFUNDA.md) — análise do algoritmo v7

---

## 2. Stack e Restrições (NÃO MUDAR SEM AUTORIZAÇÃO)

| Item | Decisão | Motivo |
|------|---------|--------|
| **Linguagem** | JavaScript vanilla (ES2020+) | App deve abrir direto no navegador, sem build |
| **CSS** | CSS puro (sem Tailwind, sem frameworks) | Tamanho mínimo do app, zero dependências |
| **PDF read** | `pdf.js` v3.11.174 via CDN | Líder de mercado, MIT |
| **PDF write** | `pdf-lib` v1.17.1 via CDN | Líder de mercado, MIT |
| **Persistência** | File System Access API | Única forma de salvar binários em pasta local no Chrome/Edge |
| **Backend** | ❌ Nenhum | Tudo client-side por requisito de privacidade |
| **Build step** | ❌ Nenhum | `git clone && abrir index.html` deve funcionar |
| **NPM/yarn** | ❌ Não usar | Nenhuma dependência local |
| **TypeScript** | ❌ Não usar | Sem build step |
| **React/Vue/Svelte** | ❌ Não usar | Sem build step |

> ⛔ **NUNCA** introduza dependências locais, build steps ou frameworks. Mudar isso é redesign de produto, não refatoração.

---

## 3. Estrutura de arquivos

```
novodiv/
├── index.html              # 🟢 O APP INTEIRO — não criar outros .html/.js/.css
├── logo.png
├── favicon.png
│
├── processos/              # 📥 ENTRADA
├── NOVOS/                  # 📥 ENTRADA
├── erros/                  # 📥 ENTRADA (casos extremos)
├── procon/                 # 📥 ENTRADA
│
├── teste1/                 # 📤 SAÍDA
│
├── docs/
│   └── LEITURA_PROFUNDA.md
│
├── README.md
├── SPEC.md
├── AGENTS.md               # este arquivo
└── LICENSE
```

**Pastas de entrada/saída** (`processos/`, `NOVOS/`, `erros/`, `procon/`, `teste1/`) são áreas de trabalho do usuário — **não versionar PDFs novos**, apenas manter como exemplos se necessário.

---

## 4. Convenção de código

### 4.1 Estilo JavaScript
- **Indentação:** 4 espaços
- **Aspas:** simples (`'`) para strings, duplas (`"`) em JSON
- **Ponto e vírgula:** sempre
- **`const` por padrão**, `let` apenas se reatribuído, **nunca** `var`
- **Arrow functions** em callbacks, **function declarations** no top-level
- **Nomes:**
  - Funções: `camelCase` (`detectInicialRange`, `isSystemPage`)
  - Constantes globais: `UPPER_SNAKE_CASE` (`START_PATTERNS`, `CLOSING_PATTERNS`)
  - Variáveis: `camelCase` descritivo (`pagesText`, `endById`)
- **Comentários:** português para lógica de negócio (é o que o usuário lê), inglês só para coisas técnicas genéricas
- **Logs de UI:** usar `slog`-like via console (`console.log` em dev, removidos em prod se aplicável)

### 4.2 Estrutura interna do `index.html`
O arquivo está organizado em seções claramente delimitadas por comentários:

```js
// ============================================================
// ALGORITHM v7   ← funções puras de detecção
// ============================================================

// ============================================================
// UI             ← handlers de DOM, drag&drop, etc.
// ============================================================
```

**Sempre preservar essa organização** ao adicionar novas funções. Se criar uma seção nova, use o mesmo padrão de delimitador.

### 4.3 Funções do algoritmo
Devem ser **puras** sempre que possível (sem efeitos colaterais, mesmo `pagesText` como input). Exceções são as funções de I/O (`splitSavePDF`, `salvar`).

### 4.4 Regex
- Adicionar padrões novos em `START_PATTERNS` / `CLOSING_PATTERNS` / `STRONG_MARKERS` (topo do script)
- **Sempre** testar regex com flag `i` (case-insensitive)
- Para padrões de fechamento, considerar `isRealClosing()` que já filtra falsos positivos
- **Nunca** usar regex com `g` em `String.match()` sem `while` loop (causa lastIndex bugs)

---

## 5. Convenção de commits

Seguimos **Conventional Commits** em português (título curto) com escopo opcional:

```text
<tipo>(<escopo>): <descrição curta>

<corpo opcional explicando o "quê" e o "porquê">

<referência a issue se aplicável>
```

### Tipos permitidos

| Tipo | Quando usar | Exemplo |
|------|-------------|---------|
| `feat` | Nova funcionalidade | `feat(algoritmo): adiciona refinamento PET3` |
| `fix` | Correção de bug | `fix(end): corrige endByContent quando OAB vem antes` |
| `docs` | Só documentação | `docs: atualiza SPEC.md com fase 0.5` |
| `refactor` | Mudança interna sem mudar comportamento | `refactor: extrai isRealClosing para função pura` |
| `test` | Adiciona/corrige testes | `test: adiciona casos de PROCON` |
| `chore` | Manutenção (deps, configs) | `chore: bump pdf.js para 3.11.200` |
| `perf` | Performance | `perf: evita extrair texto de páginas de sistema` |
| `revert` | Reverte commit anterior | `revert: feat(algoritmo) removia cobertura` |

### Exemplos válidos
```
feat(algoritmo): adiciona suporte a PET1 do eProc
fix(closing): descarta "P. Deferimento" quando vem de continuação
docs(SPEC): documenta novo score de confiança
refactor: separa detectInicialRange em 3 funções puras
```

### Regras
- **Título ≤ 72 caracteres**
- **Sem ponto final no título**
- **Corpo** justifica o "porquê" (o diff já mostra o "quê")
- **Um commit por mudança lógica** (não empacotar feat + fix)

---

## 6. Pull Requests

### 6.1 Checklist antes de abrir PR
- [ ] Mudança foi testada com pelo menos 3 PDFs reais (de `processos/`, `NOVOS/` ou `procon/`)
- [ ] Relatório gerado confere com a inspeção manual das páginas
- [ ] Não introduziu dependências locais
- [ ] Não quebrou a organização de seções do `index.html`
- [ ] Se alterou regex/padrões, atualizou `SPEC.md` (seção 5.2)
- [ ] Se alterou score, atualizou `SPEC.md` (seção 5.3)
- [ ] Se alterou UI, atualizou paleta em `SPEC.md` (seção 8.1)

### 6.2 Título do PR
Mesmo formato do commit, em modo imperativo:
```
Adicionar suporte a PET1 do eProc
Corrigir detecção de fim quando OAB vem antes
```

### 6.3 Descrição do PR
Template:
```markdown
## O que
<1-2 frases>

## Por quê
<motivação, problema que resolve>

## Como testar
<passos manuais com PDFs específicos>

## Mudanças em SPEC
<sim/não — se sim, linkar diff>
```

---

## 7. Princípios de design do algoritmo

Quando for modificar `detectInicialRange`, **siga estes princípios**:

### 7.1 Conservador > Agressivo
- Na dúvida entre dois cortes, pegue o **menor** (mais cedo termina, mais conteúdo fica no `Inicial_*.pdf`)
- É mais fácil revisar um `Inicial_*.pdf` com conteúdo a mais do que descobrir que falta um parágrafo

### 7.2 Fallback em cascata
- Sempre tenha um **plano B** quando o plano A falha
- Nunca retorne `null` sem ter esgotado as heurísticas

### 7.3 Flags > Erros
- Quando houver dúvida, **marque flag `review`** e deixe o humano decidir
- Só retorne `error` quando for impossível processar (PDF corrompido, sem camada de texto, etc.)

### 7.4 Logs são evidência
- Cada decisão de "por que este intervalo" deve ser logada no `logArea` (ou documentada nos flags retornados)
- O usuário precisa confiar no relatório

### 7.5 Sem overfitting
- Não otimize regex para 1 PDF específico
- Adicione 3+ casos de teste antes de considerar uma regex "validada"

---

## 8. Áreas de contribuição (boas primeiras tarefas)

| Tarefa | Dificuldade | Impacto |
|--------|-------------|---------|
| Adicionar mais padrões em `START_PATTERNS` (variações regionais) | 🟢 Fácil | Médio |
| Adicionar OCR fallback com Tesseract.js (opt-in) | 🟡 Médio | Alto |
| Migrar libs do CDN para local (modo offline real) | 🟡 Médio | Alto |
| Implementar export do relatório como JSON | 🟢 Fácil | Médio |
| Adicionar testes automatizados (Jest + jsdom) | 🔴 Difícil | Alto |
| Suporte a outros tipos de peça (contestação, sentença) | 🔴 Difícil | Alto |
| PWA completo (service worker, manifest) | 🟡 Médio | Médio |
| Tradução i18n (pt-BR, en, es) | 🟡 Médio | Baixo |

---

## 9. Não faça (anti-patterns)

### ⛔ NUNCA:
- **Não criar arquivos `.js`/`.css` separados** — todo código vai em `index.html`
- **Não introduzir `package.json`** — sem dependências locais
- **Não usar frameworks** (React, Vue, Svelte, Alpine) — sem build
- **Não usar TypeScript** — sem build
- **Não usar bundlers** (Webpack, Vite, Rollup) — sem build
- **Não mudar `//go:embed` ou similar** — não existe no projeto
- **Não commitar PDFs novos** em `processos/`, `NOVOS/`, `erros/`, `procon/`, `teste1/` — essas pastas são de trabalho do usuário
- **Não fazer upload para lugar nenhum** — privacidade é requisito
- **Não usar `localStorage` / `sessionStorage` / cookies** — sem persistência no browser
- **Não usar telemetria / analytics** — privacidade é requisito

### ⚠️ CUIDADO:
- **Não adicionar regex sem testar** com pelo menos 5 PDFs reais
- **Não mudar o score de confiança** sem atualizar `SPEC.md` seção 5.3
- **Não remover a flag `review`** — ela é a rede de segurança do usuário
- **Não alterar `endById` para pegar o maior candidato** — conservador é o menor
- **Não adicionar `async/await` redundante** em funções já puras
- **Não usar `var`** — sempre `const` ou `let`

---

## 10. Versionamento

- **Major (v1 → v2):** mudança incompatível de comportamento (ex: novo layout de saída, remoção de sistema suportado)
- **Minor (v1.0 → v1.1):** nova heurística, novo sistema suportado
- **Patch (v1.0.0 → v1.0.1):** bug fix, regex adicional, melhoria de score

A versão fica apenas no nome do algoritmo (ex: "Algorithm v7") e é incrementada quando há mudança significativa na heurística.

---

## 11. Quando pedir confirmação ao humano

Antes de fazer essas mudanças, **PARE e pergunte**:

1. Adicionar/remover sistema judicial suportado (PJe, Projudi, eProc, PROCON)
2. Mudar o formato de saída (`Inicial_*.pdf` / `Docs_*.pdf`)
3. Mudar o score de confiança (faixa, pesos, mapeamento)
4. Adicionar dependência externa nova (mesmo via CDN)
5. Mudar o padrão de cor/UI/UX
6. Qualquer coisa que afete o **formato do relatório** exibido
7. Qualquer coisa que afete o **fluxo de seleção de pasta**
8. Qualquer coisa que afete o **filtro de entrada** (regex de CNJ, exclusão de Inicial_*/Docs_*)

---

## 12. Recursos úteis

- **pdf.js docs:** https://mozilla.github.io/pdf.js/
- **pdf-lib docs:** https://pdf-lib.js.org/
- **File System Access API:** https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API
- **CNJ (numeração única):** https://www.cnj.jus.br/programas-e-acoes/numeracao-unica

---

*Este arquivo é lido por agentes antes de qualquer alteração. Atualize-o quando as convenções mudarem.*
