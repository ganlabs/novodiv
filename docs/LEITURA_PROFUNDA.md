# 📖 Leitura Profunda — NovoDiv

> Documento de compreensão do código do projeto **NovoDiv** (Novo Divisor de PDFs Jurídicos).
> **Atualizado após testes com ground truth (Pasta Baixados/iniciais).**

---

## 🎯 Visão Geral

O projeto é um **aplicativo single-file HTML/JavaScript puro** (sem backend, sem build) chamado **NovoDiv** (Novo Divisor) que automatiza a divisão de PDFs de processos jurídicos brasileiros. Ele recebe PDFs "integrais" (contendo todo o processo) e extrai automaticamente apenas a **Petição Inicial**, salvando o restante em um arquivo separado.

> 💡 Funciona 100% no navegador (client-side), usando as bibliotecas `pdf-lib` e `pdf.js` carregadas via CDN.

---

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│  index.html (single file, ~650 linhas)                      │
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

### 1. Reconstrução Espacial de Texto (`pdf.js`)
O maior desafio técnico do algoritmo está na forma como o `pdf.js` extrai textos: ele freqüentemente devolve as strings fragmentadas (sílabas ou pedaços de palavras). 
Se usarmos um simples `join('\n')` ou `.map(item => item.str).join(' ')`, corremos o risco de quebrar padrões de fechamento com espaços ou enter falsos (ex: `OAB \n / \n RS`).
Para resolver "falhas simples de divisão", a extração agora usa a **posição vertical (Y)** (`transform[5]`). Só ocorre quebra de linha `\n` quando a diferença na posição Y entre blocos é > 2. Isso assegura que "OAB/RS" seja lido corretamente em uma única linha.

### 2. Camada de Configuração (padrões regex)

O algoritmo usa **três famílias de padrões** declaradas no topo:

| Constante | Função |
|-----------|--------|
| `START_PATTERNS` | Detecta o **início** da petição (`EXCELENT[ÍI]SSIM[OA]`, `AO DOUTO JUIZADO`, etc.) |
| `CLOSING_PATTERNS` | Detecta o **fim** da petição (`Pede deferimento`, `VALOR DA CAUSA`, etc.) |
| `STRONG_MARKERS` | Identifica **páginas de sistema** do tribunal (cabeçalhos do PJe/Projudi/eProc) |

### 3. Funções de classificação de páginas

- **`isSystemPage(text)`** — decide se uma página é "metadado de sistema" (curta, com cabeçalho do tribunal) ou conteúdo real.
- **`isNewDocByContent(text)`** — detecta se uma página é o início de um **novo documento** dentro do processo (procuração, QR-CODE, "fls.", Carteira de Trabalho, etc.).
- **`hasPetitionEndSignal(text)`** — sinal combinado de fim: fechamento E (OAB ou Defensoria Pública).
- **`extractDocId(text)`** — extrai o **ID de documento interno** do tribunal em uma página (suporta PJe, Projudi e eProc). Retorna `[sistema, número]`.

### 4. Algoritmo Principal — `detectInicialRange(pdfBytes)` (o coração)

Este é o "Algorithm v7" que opera em **três fases sequenciais**:

#### FASE 0 — Detecção de PROCON
- Procura nas primeiras 20 páginas por marcadores específicos.
- **Heurística:** vai página a página a partir do início até encontrar nova numeração de documento ou novo doc por conteúdo.

#### FASE 0.5 — OCR de Emergência (Para processos Escaneados/Atermações)
- Se após varrer o PDF as primeiras páginas contiverem pouquíssimo texto extraído, o sistema deduz tratar-se de um PDF-imagem (como atermações e petições físicas escaneadas).
- Dispara-se o `Tesseract.js` via CDN: o app renderiza a página em um elemento `<canvas>` invisível e realiza a Leitura Óptica (OCR).
- O processo avança página a página, minimizando o impacto na memória (e só é ativado em caso de falha completa de leitura textual nativa).

#### FASE 1 — Detecção do INÍCIO da petição
- Se não é PROCON:
  1. **Pattern matching:** itera páginas pulando as de sistema, testa cada `START_PATTERNS`.
  2. **Fallback:** se nenhum padrão bate, pega a primeira página "real" com mais de 200 caracteres.
  3. **Refinamento — candidato alternativo:** se a primeira detecção tem ≤ 2 páginas, busca uma segunda ocorrência do padrão que tenha ≥ 2× mais páginas (descarta petições muito curtas que podem ser peças posteriores).
  4. **Refinamento INIC1:** se existe uma página anterior com ID `INIC1`/`INIC`/`PET1` no eProc, realinha o início para lá.
  5. **Refinamento TERMO2:** se a petição é `INIC1` única, procura uma página `TERMO2` imediatamente após (alguns processos têm termos de audiência/despacho antes da inicial, e a petição começa após eles).

#### FASE 2 — Detecção do FIM da petição

Para cada página a partir do início, coleta **5 candidatos** concorrentes:

| Candidato | Critério |
|-----------|----------|
| `endById` | Quando o ID do documento muda (ex: de `INIC1` para `PET2`) |
| `endByContent` | Quando aparece nova procuração/QR-CODE/etc., **e** a página anterior tem sinal de fim |
| `endByClosingConfirmed` | Fechamento + OAB/Defensoria + próxima página é novo doc/sistema |
| `endByClosingNoOab` | Fechamento sem OAB mas próxima página é sistema |
| `endByClosingOnly` | Fechamento + OAB/Defensoria (sem confirmação adicional) |

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

---

## 🔍 Observações Importantes (Análise Crítica e Refinamentos)

### ✅ Pontos Fortes e Correções Recentes
1. **Privacidade e Portabilidade:** Sem backends, lida com PDFs locais gigas sem lag de upload.
2. **Correção do "Split Silábico" da PDF.js:** A leitura anterior injetava `\n` indiscriminadamente, separando OAB/RS para `OAB\n/\nRS`, quebrando todas as RegEx. Isso gerava cortes em páginas esdrúxulas (ex: cortando na página 56 quando a petição acabava na 20). A reconstrução pelo eixo Y `transform[5]` resolveu 90% dos erros grosseiros.
3. **Regex Robustas e Flexíveis:** Atualizadas para suportar "EXCELENTÍSSIMA", "EXCELENTÍSSIMO", "DOUTO JUIZADO", e assinatura de "Defensora Pública" (antes exclusivas para OAB).
4. **Detecção de anexos invisíveis:** Expandido o `isNewDocByContent` para enxergar Carteira de Trabalho Digital, Cédulas de Identidade e Comprovantes, forçando o corte imediato caso a inicial não possua fecho (muito comum em anexos brutos).

### ⚠️ Limitações
1. **Petições em modo Imagem ou Iniciais inseridas Oralmente (Atermação):** Em alguns PDFs do PJe (ex: 5126846), a petição principal pode não ter texto OCRizado e ser lida apenas como a tarja do tribunal. Isso força o algoritmo a usar métricas de metadados (`endById`) que, em ausência deles, geram flags de revisão. (Solução definitiva exige OCR no client, mas custaria muito peso no app).
2. **Petições Iniciais Duplicadas:** Se o advogado anexa o mesmo PDF da inicial duas vezes ou um rascunho de rerratificação em seguida, o corte do script pode parar na primeira inicial correta, enquanto o usuário humano poderia englobar ambas em um só extraído (vide caso de `0009271-64.2026.8.05.0274`). A decisão do algoritmo de focar na menor (minimizando lixo) se provou mais segura.

---

## 💡 Resumo de uma Frase

> **NovoDiv** é um extrator offline de petições iniciais de processos jurídicos brasileiros, implementado como single-page HTML+JS que usa heurísticas regex aplicadas sob a técnica de mapeamento horizontal/vertical (eixo Y) do `pdf.js`, localizando e separando automaticamente a peça inicial com base no cabeçalho, ID do sistema e assinatura, gerando um placar de confiança para triagem de revisão humana.
