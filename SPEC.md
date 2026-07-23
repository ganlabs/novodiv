# SPEC — Especificação Técnica e Arquitetural do NovoDiv

> Especificação técnica detalhada, arquitetura do sistema, regras de negócio e especificações da interface do **NovoDiv** (Algoritmo v7 + Visualizador Canvas Interativo).

---

## 1. Visão Geral do Sistema

### 1.1 Propósito
O **NovoDiv** é uma solução *client-side* concebida para automatizar a identificação, corte e separação da **Petição Inicial** em PDFs integrais de processos judiciais e administrativos brasileiros.

### 1.2 Casos de Uso Cobertos
- **Extração Automática em Lote:** Processamento de múltiplos PDFs integrais de tribunais de justiça (PJe, Projudi, eProc) e notificações do PROCON Estadual.
- **Divisão em Dois Binários:**
  - `Inicial_{CNJ}.pdf`: Páginas correspondentes à Petição Inicial.
  - `Docs_{CNJ}.pdf`: Concatenação de todas as demais páginas do processo (peças anteriores, documentos anexos, procurações, comprovantes e certidões).
- **Auditoria & Correção Manual Visual:** Interface integrada em Canvas para navegação página a página, permitindo re-marcação de pontos de corte e re-execução instantânea de divisão.

### 1.3 Princípios Fundamentais & Restrições
- **Zero Backend / Client-Side Total:** Operações de parsing de texto, renderização em Canvas, edição de PDF e gravação em disco acontecem 100% no navegador do usuário.
- **Privacidade & LGPD:** Nenhum dado pessoal, número de processo ou conteúdo de documento sai da máquina local.
- **Zero Build Step:** Todo o código reside em um único arquivo (`index.html`), rodando diretamente sem necessidade de compilação, bundlers (`webpack`/`vite`), gerenciadores de pacotes (`npm`/`yarn`) ou TypeScript.

---

## 2. Requisitos de Sistema

### 2.1 Requisitos Funcionais (RF)

| ID | Descrição | Implementação / Componente |
|----|-----------|----------------------------|
| **RF-01** | Receber arquivos PDF por *Drag & Drop* ou seletor de arquivos. | `dropZone`, `fileInput` |
| **RF-02** | Filtrar entradas e validar padrão de numeração CNJ (`NNNNNNN-NN.YYYY.J.TR.OOOO`). | `handleFiles()`, `cnjFromName()` |
| **RF-03** | Identificar o intervalo `[startPage, endPage]` da Petição Inicial via heurística multi-fase. | `detectInicialRange()` (Algorithm v7) |
| **RF-04** | Executar fallback de OCR em PDFs digitalizados (sem camada de texto nativa). | `Tesseract.js` v5 em `detectInicialRange()` |
| **RF-05** | Gerar e salvar fisicamente os arquivos `Inicial_{CNJ}.pdf` e `Docs_{CNJ}.pdf`. | `splitSavePDF()`, `salvar()` via `showDirectoryPicker()` |
| **RF-06** | Apresentar métricas de execução em tempo real (Total, OK, Revisar, Erros, Barra de Progresso). | Dashboard Stepper (Passos 1, 2, 3) |
| **RF-07** | Exibir visualizador visual em Canvas com controles de Zoom e Rolagem Fluida. | `customViewer`, `renderPage()`, `getFitScale()` |
| **RF-08** | Permitir ajuste manual do corte (`Marcar Início` / `Marcar Fim` / `Dividir`). | `openManualSplit()`, `btnManualSplit` listener |
| **RF-09** | Exportar relatório de auditoria dos processos em formato CSV com UTF-8 BOM. | `exportToCsv()` |

### 2.2 Requisitos Não-Funcionais (RNF)

| ID | Descrição | Métrica / Critério de Aceite |
|----|-----------|-------------------------------|
| **RNF-01** | **Privacidade:** Zero tráfego de rede durante o processamento de documentos. | Verificável em DevTools (Network Tab silenciosa). |
| **RNF-02** | **Desempenho:** Processar PDF de 100 páginas em menos de 5 segundos (com camada de texto). | Testado em hardware i5 / 8GB RAM. |
| **RNF-03** | **Precisão:** Taxa de acertabilidade da detecção automática ≥ 85%. | Validado contra a suíte de testes em `processos/`. |
| **RNF-04** | **Navegação Suave:** Evitar saltos múltiplos de página durante a rolagem rápida de mouse/trackpad. | Engine de *Wheel Throttling* ajustado com cooldown de **120ms**. |
| **RNF-05** | **Compatibilidade de Nomenclatura:** Suporte a arquivos com extensões de zona (ex: `Zone.Identifier` do Windows). | Filtro estrito de arquivos `.pdf`. |

---

## 3. Arquitetura de Software e Componentes

### 3.1 Visão Geral do Arquivo Unificado (`index.html`)

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                               index.html                                │
│                                                                         │
│  ┌───────────────────────┐  ┌────────────────────────────────────────┐  │
│  │ UI & DOM Layer        │  │ Algorithm v7 & OCR Engine              │  │
│  │ - Stepper Steps 1..3  │  │ - START_PATTERNS / CLOSING_PATTERNS    │  │
│  │ - Stats Dashboard     │  │ - detectInicialRange() (Fases 0..2)    │  │
│  │ - Modal Relatório     │  │ - Tesseract.js Worker Fallback         │  │
│  └───────────────────────┘  └────────────────────────────────────────┘  │
│             │                                   │                       │
│             ▼                                   ▼                       │
│  ┌───────────────────────┐  ┌────────────────────────────────────────┐  │
│  │ Custom Viewer Canvas  │  │ I/O Layer (File System Access API)     │  │
│  │ - Fit Scale Engine    │  │ - showDirectoryPicker()                │  │
│  │ - 120ms Wheel Throttle│  │ - pdf-lib (splitSavePDF)               │  │
│  │ - Manual Marking      │  │ - Export CSV (UTF-8 BOM)               │  │
│  └───────────────────────┘  └────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Bibliotecas Externas (CDN)

1. **`pdf-lib@1.17.1`** (`pdf-lib.min.js`):
   - Responsável por carregar o documento binário, copiar o conjunto de páginas desejado (`copyPages`) e gravar os novos documentos PDF preservando a qualidade e metadados originais.
2. **`pdf.js@3.11.174`** (`pdf.min.js` e `pdf.worker.min.js`):
   - Responsável pela extração estruturada do conteúdo de texto por página (`getTextContent`) e pela renderização em Canvas HTML5 (`page.render`).
3. **`Tesseract.js@5`** (`tesseract.min.js`):
   - Executa OCR de emergência no navegador via Web Worker, processando imagens de páginas renderizadas temporariamente em Canvas.

---

## 4. Algoritmo de Detecção (Algorithm v7)

### 4.1 Fases do Algoritmo

```text
┌─────────────────────────────────────────────────────────────────┐
│ FASE 0: PROCON Check                                            │
│ Verifica termos de notificação administrativa nas pág 1..20.    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ FASE 0.5: OCR Fallback Check                                    │
│ Se total_chars(1..10) < 500 → Ativa Tesseract.js Worker.        │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ FASE 1: Detecção de Início (startPage)                          │
│ 1. Match de START_PATTERNS (vocativos jurídicos).               │
│ 2. Fallback: primeira página com > 200 caracteres úteis.        │
│ 3. Refinamento pattern_alt: corrige petições curtas (≤ 2 pág).  │
│ 4. Refinamento INIC1 / TERMO2: adequação para eProc / PJe.      │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ FASE 2: Detecção de Fim (endPage)                               │
│ Avalia 5 concorrentes e toma endPage = min(candidatos):         │
│ - endById: alteração no número/código do documento no sistema.  │
│ - endByContent: início de procuração, certidão ou relatório.   │
│ - endByClosingConfirmed: fechamento + OAB + próximo é sistema. │
│ - endByClosingNoOab: fechamento formal + próximo é sistema.     │
│ - endByClosingOnly: fechamento + OAB.                           │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ FASE 3: Cálculo de Confiança (Score 0..5) & Flags               │
│ Atribui pontuação e define a flag 'review' (se score < 3).      │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Regras de Regex e Filtragem

#### Vocativos de Início (`START_PATTERNS`)
```javascript
/EXCELENT[ÍI]SSIM[OA]\b/i,
/AO\s+DOUTO\s+JU[ÍI]ZO/i, /AO\s+JU[ÍI]ZO/i,
/AO\s+(?:DOUTO\s+)?JU[ÍI]ZADO\b/i, /EXMO\.?\s*(?:SR\.?)\s+DR\.?\s+JU[ÍI]Z/i,
/JU[ÍI]ZO\s+DE\s+DIREITO\s+DA\s+VARA/i,
/MERITÍSSIMO/i, /MERITISSIMO/i,
/EXMO\s*\(A\)/i, /EXMO\.?\s*(?:\(A\)\s*)?DR?\.?\s*JU[ÍI]Z/i,
/DEFENSORIA\s+PÚBLICA/i, /DEFENSORIA\s+PUBLICA/i
```

#### Fechamentos Validados (`isRealClosing`)
Evita falsos positivos em citações de jurisprudência verificando se a palavra seguinte não é um conectivo de continuidade (`da`, `do`, `de`, `a`, `o`).
```javascript
/[Pp]ede\s+[Dd]eferimento/, /[Pp]ede[- ]*se\s+[Dd]eferimento/,
/[Pp]ede\s+e\s+(?:espera|agna)\s+(?:o|por)?\s*[Dd]eferimento/,
/[Cc]onfia\s+(?:e\s+)?(?:espera|agna)\s+(?:o|por)?\s*[Dd]eferimento/,
/[Cc]onfia\s+no\s+[Dd]eferimento/, /P\.?\s*[Dd]eferimento/,
/[Tt]ermos\s+em\s+que/, /[Nn]estes\s+[Tt]ermos/, /[Nn]esses\s+[Tt]ermos/,
/[Dd]á-se\s+(?:a|à)\s+causa/, /[Ss]endo\s+assim/, /consumidora\s+requer/i, /VALOR\s+DA\s+CAUSA/
```

#### Identificação de Novos Documentos por Conteúdo (`isNewDocByContent`)
Identifica o início de documentos anexos subsequentes através do cabeçalho da página:
- Procurações (`PROCURAÇÃO`, `P R O C U R A Ç Ã O`)
- Consultas a órgãos de crédito (`SERASA`, `SPC Brasil`, `CREDNET`)
- Relatórios ou calculadoras do INSS / Crednet
- Páginas de separação do tribunal / Capa de processo
- Documentos de identificação (`RG`, `Carteira de Trabalho`, `COMPROVANTE`, `CERTIDÃO`)

### 4.3 Matriz de Confiança e Pontuação

```text
Pontuação de Início:
+2 : startMethod === 'pattern' OU 'inic1'
+1 : startMethod === 'pattern_alt', 'termo2' OU 'procon' (adiciona flag 'review')
+0 : startMethod === 'fallback' OU 'ocr_pattern' (adiciona flag 'review')

Pontuação de Fim:
+2 : endMethod === 'id', 'close_conf' OU 'content'
+1 : endMethod === 'close_no_oab' (adiciona flag 'review') OU 'close_only'
+0 : endMethod === 'fallback' (adiciona flag 'review')

Penalidades de Tamanho:
-1 : Se (endPage - startPage + 1) < 3 páginas
-1 : Se (endPage - startPage + 1) > 60 páginas

Score Final: clamp(score, 0, 5)
Se score < 3 → Ativa a flag de auditoria 'review' (Revisar)
```

---

## 5. Visualizador Customizado & Sub-sistema Canvas

### 5.1 Mecânica do Visualizador Customizado (`customViewer`)

Para permitir a auditoria e correção manual de intervalos sem depender de leitores externos, o aplicativo implementa uma interface em Canvas controlada por estado em memória:

```javascript
let currentDoc = null;        // Documento PDF lido via pdf.js
let currentPageNum = 1;       // Página ativa em exibição
let isRenderingPage = false;  // Trava de renderização concorrente
let customScale = null;       // Fator de zoom customizado (null = Fit)
```

### 5.2 Algoritmo de Zoom Fit (`getFitScale`)
Calcula a escala exata para ajustar a página inteira à área visível da viewport, respeitando margens de interface:

```javascript
function getFitScale() {
    if (!unscaledViewport) return 1.0;
    const cw = customViewer.clientWidth - 40;  // Largura visível menos padding
    const ch = customViewer.clientHeight - 80; // Altura visível menos barra superior
    const scaleW = cw / unscaledViewport.width;
    const scaleH = ch / unscaledViewport.height;
    return Math.min(scaleW, scaleH);
}
```

### 5.3 Rolagem Fluida & Cooldown de Roda do Mouse (`Wheel Throttling`)
Dispositivos modernos (mouses de alta sensibilidade como a linha Logitech MX e trackpads com rolagem inercial) emitem dezenas de eventos `wheel` em um único movimento.

Para evitar saltos indesejados de múltiplas páginas, o aplicativo utiliza um throttle com trava de **120ms**:

```javascript
let lastWheelTime = 0;

customViewer.addEventListener('wheel', async (e) => {
    if (isRenderingPage || !currentDoc) return;
    
    // Cooldown de 120ms: sweet spot entre fluidez inercial e precisão de página
    if (Date.now() - lastWheelTime < 120) return;
    
    const atTop = customViewer.scrollTop <= 10;
    const atBottom = Math.ceil(customViewer.scrollTop + customViewer.clientHeight) >= customViewer.scrollHeight - 10;
    
    if (e.deltaY > 0 && atBottom && currentPageNum < currentDoc.numPages) {
        e.preventDefault();
        lastWheelTime = Date.now();
        await renderPage(currentPageNum + 1);
        customViewer.scrollTop = 0;
    } else if (e.deltaY < 0 && atTop && currentPageNum > 1) {
        e.preventDefault();
        lastWheelTime = Date.now();
        await renderPage(currentPageNum - 1);
        customViewer.scrollTop = customViewer.scrollHeight; 
    }
});
```

---

## 6. Modelo de Dados e Exportação

### 6.1 Estrutura do Relatório em Memória (`reportData`)

```typescript
interface ReportEntry {
  file: string;          // Nome original do arquivo (ex: Integral_0003180-66.2026.8.05.0141.pdf)
  cnj: string | null;    // Número CNJ extraído
  start?: number;        // Página inicial (1-based)
  end?: number;          // Página final (1-based)
  total?: number;        // Total de páginas do arquivo original
  pages?: number;        // Quantidade de páginas da Petição Inicial (end - start + 1)
  flags?: string[];      // Marcadores ('PROCON', 'TERMO2', 'OCR', 'review', 'Manual')
  needsReview: boolean;  // Se requer atenção humana
  error?: string | null; // Mensagem de exceção caso falhe
}
```

### 6.2 Especificação da Exportação CSV

- **Nome do Arquivo:** `NovoDiv_Relatorio_{YYYY-MM-DDTHH-mm-ss}.csv`
- **Encoding:** UTF-8 com BOM (`\uFEFF`) para compatibilidade com Microsoft Excel e Google Sheets.
- **Delimitador:** Ponto e vírgula (`;`).
- **Campos Exportados:** `Arquivo`, `CNJ`, `Pagina_Inicial`, `Pagina_Final`, `Total_Paginas_Peticao`, `Status`, `Precisa_Revisao`, `Flags`, `Mensagem_Erro`.

---

## 7. Registro de Decisões Arquiteturais (ADR)

### ADR-01: Manutenção do Modelo Single-File (`index.html`)
- **Status:** Aprovado / Mantido.
- **Contexto:** Necessidade de rodar a aplicação em máquinas corporativas restritas sem permissão de administrador ou instalação de Node.js/Python.
- **Decisão:** Manter 100% da lógica (HTML, CSS e JavaScript) em `index.html`.
- **Consequência:** Facilidade extrema de implantação (`git clone` ou copiar arquivo), mas exige disciplina estrita na organização de seções do código.

### ADR-02: Uso da File System Access API para I/O Local
- **Status:** Aprovado.
- **Contexto:** O navegador padrão impede salvar arquivos em massa em uma pasta específica sem disparar dezenas de diálogos de download.
- **Decisão:** Utilizar `window.showDirectoryPicker()` para obter permissão de escrita direta na pasta de destino escolhida pelo usuário.
- **Consequência:** Requer uso de navegadores baseados em Chromium (Chrome/Edge/Opera).

### ADR-03: Redução do Cooldown do Scroll para 120ms
- **Status:** Aprovado.
- **Contexto:** O cooldown anterior de 400ms tornava a navegação travada em mouses de alta velocidade e trackpads com rolagem contínua.
- **Decisão:** Reduzir a trava de `wheel` para 120ms.
- **Consequência:** Alcançou a taxa ideal de transição (até ~8 páginas/segundo em rolagem contínua) sem disparar múltiplos renderizamentos assíncronos simultâneos no Canvas.

### ADR-04: Estrutura da Pasta `.untracked/`
- **Status:** Aprovado.
- **Contexto:** PDFs de testes reais do usuário contêm dados jurídicos e não devem ser versionados no Git.
- **Decisão:** Criar a regra `.untracked/` no arquivo `.gitignore` para alocar pastas locais como `Baixados/`.
- **Consequência:** Repositório limpo e em conformidade com as políticas de privacidade.

---

*Especificação mantida e atualizada em `SPEC.md`.*
