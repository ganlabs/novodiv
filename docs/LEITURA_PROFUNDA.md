# 📖 Leitura Profunda — NovoDiv

> Documento de compreensão do código do projeto **NovoDiv** (Novo Divisor de PDFs Jurídicos).

---

## 🎯 Visão Geral

O projeto é um **aplicativo single-file HTML/JavaScript puro** (sem backend, sem build) chamado **NovoDiv** (Novo Divisor) que automatiza a divisão de PDFs de processos jurídicos brasileiros. Ele recebe PDFs "integrais" (contendo todo o processo) e extrai automaticamente apenas a **Petição Inicial**, salvando o restante em um arquivo separado.

> 💡 Funciona 100% no navegador (client-side), usando as bibliotecas `pdf-lib` e `pdf.js` carregadas via CDN.

---

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│  index.html (single file, ~640 linhas)                      │
│                                                              │
│  ┌──────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │  UI Layer    │  │ Algorithm v7    │  │  I/O Layer     │  │
│  │  (HTML+CSS)  │←→│ (pure JS)       │←→│  (File System  │  │
│  │              │  │                 │  │   Access API)  │  │
│  └──────────────┘  └─────────────────┘  └────────────────┘  │
│         ↓                    ↓                     ↓         │
│    drag&drop          detectInicialRange()    showDirectory  │
│    progress bar       (FASE 0, 1, 2)           Picker()      │
│    relatório          splitSavePDF()           salvar()      │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧩 Estrutura Lógica do Código

### 1. Camada de Configuração (padrões regex)

O algoritmo usa **três famílias de padrões** declaradas no topo:

| Constante | Função |
|-----------|--------|
| `START_PATTERNS` | Detecta o **início** da petição (vocativos formais: "EXCELENTÍSSIMO", "EXMO. SR. DR. JUIZ", "AO DOUTO JUÍZO", etc.) |
| `CLOSING_PATTERNS` | Detecta o **fim** da petição ("Pede deferimento", "Termos em que", "Valor da causa", etc.) |
| `STRONG_MARKERS` | Identifica **páginas de sistema** do tribunal (cabeçalhos do PJe/Projudi/eProc) |

### 2. Funções de classificação de páginas

- **`isSystemPage(text)`** — decide se uma página é "metadado de sistema" (curta, com cabeçalho do tribunal) ou conteúdo real.
- **`isNewDocByContent(text)`** — detecta se uma página é o início de um **novo documento** dentro do processo (procuração, QR-CODE, "fls.", etc.).
- **`isRealClosing(text)`** — valida se um padrão de fechamento é genuíno (descartando falsos positivos como "P. Deferimento" que sejam continuação de frase).
- **`hasPetitionEndSignal(text)`** — sinal combinado de fim: fechamento + OAB + valor da causa + fls.
- **`extractDocId(text)`** — extrai o **ID de documento interno** do tribunal em uma página (suporta PJe, Projudi e eProc). Retorna `[sistema, número]`.

### 3. Algoritmo Principal — `detectInicialRange(pdfBytes)` (o coração)

Este é o "Algorithm v7" mencionado no comentário. Ele opera em **três fases sequenciais**:

#### FASE 0 — Detecção de PROCON
- Procura nas primeiras 20 páginas por marcadores específicos: `TERMO DE NOTIFICAÇÃO`, `PROCON ESTADUAL`, `CONVÊNIO ENTRE O PROCON`, etc.
- Se encontra, define `proconStart` e marca a flag `PROCON` (esses processos têm uma estrutura diferente).
- **Heurística específica PROCON:** vai página a página a partir do início até encontrar nova numeração de documento ou novo doc por conteúdo.

#### FASE 1 — Detecção do INÍCIO da petição
- Se não é PROCON:
  1. **Pattern matching:** itera páginas pulando as de sistema, testa cada `START_PATTERNS`. Método = `'pattern'`.
  2. **Fallback:** se nenhum padrão bate, pega a primeira página "real" com mais de 200 caracteres. Método = `'fallback'`.
  3. **Refinamento — candidato alternativo:** se a primeira detecção tem ≤ 2 páginas, busca uma segunda ocorrência do padrão que tenha ≥ 2× mais páginas (descarta petições muito curtas que podem ser peças posteriores). Método = `'pattern_alt'`.
  4. **Refinamento INIC1:** se existe uma página anterior com ID `INIC1`/`INIC`/`PET1` no eProc, realinha o início para lá. Método = `'inic1'`.
  5. **Refinamento TERMO2:** se a petição é `INIC1` única, procura uma página `TERMO2` imediatamente após (alguns processos têm termos de audiência/despacho antes da inicial, e a petição começa após eles). Método = `'termo2'`.

#### FASE 2 — Detecção do FIM da petição

Para cada página a partir do início, coleta **5 candidatos** concorrentes:

| Candidato | Critério |
|-----------|----------|
| `endById` | Quando o ID do documento muda (ex: de `INIC1` para `PET2`) |
| `endByContent` | Quando aparece nova procuração/QR-CODE/etc., **e** a página anterior tem sinal de fim |
| `endByClosingConfirmed` | Fechamento + OAB + próxima página é novo doc/sistema |
| `endByClosingNoOab` | Fechamento sem OAB mas próxima página é sistema |
| `endByClosingOnly` | Fechamento + OAB (sem confirmação adicional) |

Depois pega o **menor número de página** entre os candidatos (mais conservador, evita cortar antes do fim real).

#### Score de Confiança (0-5)

```
+2 se start = 'pattern' ou 'inic1'
+1 se start = 'pattern_alt', 'termo2' ou 'procon'
+0 se start = 'fallback'

+2 se end = 'id', 'close_conf' ou 'content'
+1 se end = 'close_no_oab' ou 'close_only'
+0 se end = 'fallback'

-1 se total de páginas < 3  (petição suspeita de curta)
-1 se total de páginas > 60 (suspeita de grande demais)
```

O score é mapeado para "acurácia" exibida no relatório: `{5:98%, 4:85%, 3:65%, 2:40%, 1:20%, 0:5%}`. Caso o score seja < 3, marca flag `review` para revisão humana.

### 4. I/O — `splitSavePDF`

Carrega o PDF com `pdf-lib`, copia as páginas do intervalo `[start, end]` para um novo PDF nomeado `Inicial_{CNJ}.pdf`, e junta **antes + depois** em `Docs_{CNJ}.pdf`. A escrita usa a **File System Access API** (`showDirectoryPicker` + `getFileHandle` + `createWritable`) — por isso o navegador pede permissão de pasta.

### 5. UI/UX

- **Drag & drop** ou clique para selecionar arquivos.
- **Filtragem de entrada:** ignora arquivos que começam com `Inicial` ou `Docs` (já processados) e exige o **CNJ** (formato `NNNNNNN-NN.YYYY.J.TR.OOOO`) no nome do arquivo.
- **Progress bar** + **log em tempo real** (estilo terminal).
- **Relatório final** com tabela: arquivo, CNJ, intervalo de páginas, acurácia estimada, flags e status (✅ OK / 👁️ Revisar / ❌ Erro).

---

## 🔄 Fluxo de Execução Resumido

```
1. Usuário arrasta PDFs integrais
         ↓
2. Filtra por nome (precisa ter CNJ; ignora "Inicial_*" e "Docs_*")
         ↓
3. Usuário escolhe pasta de destino (File System Access API)
         ↓
4. Para cada PDF:
   ├─ pdf.js extrai texto de todas as páginas
   ├─ detectInicialRange() → {start, end, confidence, flags}
   └─ splitSavePDF() → grava Inicial_{CNJ}.pdf + Docs_{CNJ}.pdf
         ↓
5. Renderiza relatório com totais e badges de revisão
```

---

## 🎨 Design Visual

- **Paleta:** dourado (`#FFC107`) + preto/cinza escuro, com **background radial** (efeito degradê do dourado para o escuro).
- **CSS puro** (sem Tailwind), com classes utilitárias simples (`.flex`, `.mt-15`, `.text-small`).
- **Logo "Gondim"** no topo (imagem PNG externa).
- **Cards brancos** com sombra sobre o fundo escuro.

---

## 📁 Estrutura de Arquivos do Projeto

```
novodiv/
├── index.html              # ← O APP INTEIRO (HTML + CSS + JS)
├── favicon.png
├── logo.png
├── processos/              # PDFs integrais de entrada (entrada principal)
├── NOVOS/                  # PDFs integrais novos a processar
├── erros/                  # PDFs integrais com problemas (gera "erros" no log)
├── procon/                 # PDFs específicos de PROCON (entrada alternativa)
└── teste1/                 # ← SAÍDA: PDFs já divididos (Inicial_* + Docs_*)
```

> O padrão de nomes de saída é `Inicial_{CNJ}.pdf` e `Docs_{CNJ}.pdf`, onde `{CNJ}` é o número do processo extraído do nome do arquivo original. Isso explica por que `teste1/` contém muitos `Docs_*.pdf` mas não `Inicial_*.pdf` — provavelmente só os "Docs" foram commitados para versionamento, ou os "Inicial" foram gerados separadamente.

---

## 🔍 Observações Importantes

### ✅ Pontos Fortes
1. **Auto-contido:** funciona offline (exceto pelo carregamento inicial das libs via CDN).
2. **Privacidade:** "Os arquivos não saem do seu computador" — tudo client-side.
3. **Suporta múltiplos sistemas judiciais:** PJe, Projudi, eProc (via `extractDocId`).
4. **Detecção específica de PROCON** com fluxo separado.
5. **Sistema de flags e confidence score** para triagem de revisão humana.
6. **Fallbacks em cascata** (pattern → fallback → refinamentos).

### ⚠️ Pontos de Atenção / Possíveis Melhorias
1. **Dependência de CDN** — se cair a internet, não funciona. Poderia usar `pdf-lib` e `pdf.js` locais.
2. **Regex frágeis** — os padrões de início (`EXCELENTÍSSIMO`) podem falhar em petições digitalizadas (imagem) já que o app só lê texto, não faz OCR.
3. **File System Access API** só funciona no Chrome/Edge — Firefox e Safari não suportam.
4. **Não há testes automatizados** — a verificação é manual via "teste1/".
5. **Código monolítico** — 640 linhas em um único `index.html` dificulta manutenção.
6. **Sem persistência de estado** — se a aba for fechada, o relatório é perdido.
7. **O algoritmo v7** sugere que houve iterações anteriores (v1-v6) que não estão neste repo.

---

## 💡 Resumo de uma Frase

> **NovoDiv** é um extrator offline de petições iniciais de processos jurídicos brasileiros, implementado como single-page HTML+JS que usa heurísticas regex + sinais estruturais (IDs PJe/Projudi/eProc) para localizar e separar automaticamente a peça inicial do restante do PDF integral, com um sistema de pontuação de confiança que marca casos duvidosos para revisão humana.

---

*Documento gerado a partir de leitura profunda do código fonte em `/home/passoz/dev/gan/novodiv/index.html`.*
