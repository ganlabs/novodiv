# SPEC — Especificação Técnica do NovoDiv

> Especificação técnica do **NovoDiv**, divisor automático de Petições Iniciais em PDFs integrais de processos jurídicos brasileiros.

---

## 1. Visão Geral

### 1.1 Propósito
Automatizar a extração da Petição Inicial de PDFs integrais de processos judiciais brasileiros, evitando trabalho manual de identificar o intervalo de páginas e dividir o arquivo.

### 1.2 Usuários-alvo
- **Advogados e equipes jurídicas** que recebem lotes de PDFs integrais e precisam apenas das iniciais para análise prévia, distribuição ou protocolo.
- **Operadores de PROCON** lidando com procedimentos administrativos.

### 1.3 Não-objetivos
- ❌ OCR de PDFs escaneados (apenas PDFs com camada de texto)
- ❌ Suporte a formatos que não sejam PDF
- ❌ Upload para servidores (toda operação é local)
- ❌ Suporte a Firefox/Safari (depende de File System Access API)

---

## 2. Requisitos Funcionais

| ID | Requisito | Prioridade |
|----|-----------|------------|
| **RF-01** | Aceitar múltiplos PDFs integrais via drag & drop ou seletor de arquivo | Alta |
| **RF-02** | Filtrar entradas por nome (deve conter CNJ no formato `NNNNNNN-NN.YYYY.J.TR.OOOO`) | Alta |
| **RF-03** | Detectar automaticamente o intervalo de páginas da Petição Inicial | Alta |
| **RF-04** | Gerar 2 arquivos: `Inicial_{CNJ}.pdf` e `Docs_{CNJ}.pdf` | Alta |
| **RF-05** | Exibir relatório com confiança (0-100%) e flag de revisão para casos duvidosos | Alta |
| **RF-06** | Suportar PDFs dos sistemas: PJe, Projudi, eProc | Alta |
| **RF-07** | Suportar PDFs de PROCON Estadual com fluxo de detecção dedicado | Média |
| **RF-08** | Mostrar log em tempo real do processamento | Média |
| **RF-09** | Exigir seleção de pasta de destino antes de processar | Alta |
| **RF-10** | Calcular `endPage = min(endById, endByContent, endByClosingConfirmed, endByClosingNoOab, endByClosingOnly)` | Alta |

---

## 3. Requisitos Não-Funcionais

| ID | Requisito | Métrica |
|----|-----------|---------|
| **RNF-01** | Privacidade total: nenhum byte sai do cliente | Verificável: zero requests de rede após carregar libs |
| **RNF-02** | Processar PDF de 100 páginas em < 10s | Em hardware modesto (i5/8GB) |
| **RNF-03** | Taxa de detecção correta ≥ 85% no conjunto de treinamento | Mensurável via revisão manual |
| **RNF-04** | Funcionar offline após primeiro carregamento | PWA-ready (futuro) |
| **RNF-05** | UI responsiva (desktop ≥ 1024px) | Prioridade: desktop |
| **RNF-06** | Sem dependências locais de build | `git clone && abrir index.html` |

---

## 4. Arquitetura

### 4.1 Diagrama de componentes

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

### 4.2 Dependências externas (CDN)
- `pdf-lib@1.17.1` → escrita/edição de PDFs
- `pdf.js@3.11.174` → leitura/extração de texto
- `pdf.worker.min.js` → worker do pdf.js (off-main-thread)

### 4.3 Estado em memória
```
filesToProcess: File[]              # PDFs selecionados pelo usuário
report: Array<{                    # resultado do processamento
  file: string
  cnj: string | null
  start: number | null
  end: number | null
  pages: number
  confidence: number               # 0-5
  flags: string[]                  # 'PROCON' | 'review' | 'TERMO2' | ...
  needsReview: boolean
  error?: string
}>
```

---

## 5. Algoritmo de Detecção

### 5.1 Visão geral das fases

```
FASE 0: PROCON?
   └─ Sim → fluxo específico (vai até primeira troca de ID ou novo doc)
   └─ Não → FASE 1

FASE 1: Encontrar INÍCIO
   1. Pattern matching com START_PATTERNS
   2. Fallback (primeira página com > 200 chars)
   3. Refinamento pattern_alt (se candidato tem ≤ 2 páginas)
   4. Refinamento INIC1 (se existe INIC1/INIC/PET1 anterior)
   5. Refinamento TERMO2 (se INIC1 é único e há TERMO2+ após)

FASE 2: Encontrar FIM
   Coleta 5 candidatos concorrentes:
   - endById:            mudança de ID de doc
   - endByContent:       novo doc por conteúdo + sinal de fim anterior
   - endByClosingConfirmed: fechamento + OAB + próxima é novo/sistema
   - endByClosingNoOab:  fechamento + próxima é sistema
   - endByClosingOnly:   fechamento + OAB (sem confirmação)
   
   endPage = min(candidatos)
```

### 5.2 Padrões regex

**START_PATTERNS** (vocativos formais):
```js
/EXCELENTÍSSIMO/i, /EXCELENTISSIMO/i,
/AO\s+DOUTO\s+JU[ÍI]ZO/i, /AO\s+JU[ÍI]ZO/i,
/AO\s+JU[ÍI]ZADO/i, /EXMO\.?\s*(?:SR\.?)\s+DR\.?\s+JU[ÍI]Z/i,
/JU[ÍI]ZO\s+DE\s+DIREITO\s+DA\s+VARA/i,
/MERITÍSSIMO/i, /MERITISSIMO/i,
/EXMO\s*\(A\)/i, /EXMO\.?\s*(?:\(A\)\s*)?DR?\.?\s*JU[ÍI]Z/i,
/DEFENSORIA\s+PÚBLICA/i, /DEFENSORIA\s+PUBLICA/i,
```

**CLOSING_PATTERNS** (fechamentos de petição):
```js
/[Pp]ede\s+[Dd]eferimento/,
/[Pp]ede[- ]*se\s+[Dd]eferimento/,
/[Pp]ede\s+e\s+(?:espera|agna)\s+(?:o|por)?\s*[Dd]eferimento/,
/[Cc]onfia\s+(?:e\s+)?(?:espera|agna)\s+(?:o|por)?\s*[Dd]eferimento/,
/[Cc]onfia\s+no\s+[Dd]eferimento/,
/P\.?\s*[Dd]eferimento/,
/[Tt]ermos\s+em\s+que/, /[Nn]estes\s+[Tt]ermos/, /[Nn]esses\s+[Tt]ermos/,
/[Dd]á-se\s+(?:a|à)\s+causa/, /[Ss]endo\s+assim/,
/consumidora\s+requer/i, /VALOR\s+DA\s+CAUSA/,
```

**STRONG_MARKERS** (cabeçalhos de sistema):
```js
'PÁGINA DE SEPARAÇÃO', 'Gerada automaticamente pelo sistema',
'Nº do processo', 'Data de autuação',
'Procedimento Comum', 'PROCEDIMENTO DO', 'JEF Previdenciária',
'Benefício Prev.', 'Classe:', 'Órgão julgador:', 'Assunto:', 'Número:',
```

### 5.3 Sistema de pontuação

```
+2 se start = 'pattern' | 'inic1'
+1 se start = 'pattern_alt' | 'termo2' | 'procon'  (+ flag 'review')
+0 se start = 'fallback'                           (+ flag 'review')

+2 se end = 'id' | 'close_conf' | 'content'
+1 se end = 'close_no_oab' | 'close_only'         (+ flag 'review' se 'close_no_oab')
+0 se end = 'fallback'                             (+ flag 'review')

-1 se endPage - startPage + 1 < 3
-1 se endPage - startPage + 1 > 60

score = clamp(score, 0, 5)
if score < 3 → flag 'review'
```

**Mapeamento score → acurácia exibida:**
| Score | Acurácia |
|-------|----------|
| 5 | 98% |
| 4 | 85% |
| 3 | 65% |
| 2 | 40% |
| 1 | 20% |
| 0 | 5% |

---

## 6. Formato dos arquivos de saída

### 6.1 Convenção de nomes
```
Inicial_{CNJ}.pdf      ← páginas [start..end] da petição inicial
Docs_{CNJ}.pdf         ← concatenação de [1..start-1] + [end+1..total]
```

**Exemplo:**
```
Entrada:  Integral_0801572-77.2026.8.10.0038.pdf
Saída:    Inicial_0801572-77.2026.8.10.0038.pdf
          Docs_0801572-77.2026.8.10.0038.pdf
```

### 6.2 Validação de CNJ
Regex: `/(\d{7}-\d{2}\.\d{4}\.\d\.\d{2}\.\d{4})/`

| Componente | Significado |
|------------|-------------|
| `\d{7}-\d{2}` | Número sequencial + dígito verificador |
| `\.\d{4}` | Ano do ajuizamento |
| `\.\d` | Segmento do poder judiciário |
| `\.\d{2}` | Tribunal |
| `\.\d{4}` | Unidade de origem |

---

## 7. Modelo de dados do relatório

```ts
type ReportEntry = {
  file: string                    // nome original
  cnj: string | null              // CNJ extraído do nome
  start: number | null            // página inicial da petição (1-indexed)
  end: number | null              // página final da petição
  pages: number                   // end - start + 1
  confidence: number              // 0-5
  flags: Flag[]                   // 'PROCON' | 'TERMO2' | 'review' | ...
  needsReview: boolean            // computed: flags.includes('review')
  error?: string                  // mensagem se falhou
}

type Flag = 'PROCON' | 'TERMO2' | 'review' | string
```

---

## 8. UI/UX

### 8.1 Paleta de cores
| Token | Hex | Uso |
|-------|-----|-----|
| `--gold` | `#FFC107` | Botões primários, ícones |
| `--dark-gold` | `#FFA000` | Hover |
| `--black` | `#1A1A1A` | Texto |
| `--gray` | `#666` | Texto secundário |
| Background | `radial-gradient(circle at 60% 100%, #d4b52b 0%, #685a21 40%, #2f2a18 100%)` | Body |

### 8.2 Estados visuais do relatório
| Badge | Cor | Significado |
|-------|-----|-------------|
| `✅ OK` | Verde (`#28a745`) | Confidence ≥ 3, sem flags críticas |
| `👁️ Revisar` | Amarelo (`#fd7e14`) | Confidence < 3 ou flag `review` |
| `❌ Erro` | Vermelho (`#dc3545`) | Falha no processamento |

### 8.3 Barra de confiança
- Verde (`#28a745`): alta (≥ 4)
- Dourado (`#FFC107`): média (= 3)
- Vermelho (`#dc3545`): baixa (≤ 2)

---

## 9. Limitações Conhecidas

| ID | Limitação | Workaround |
|----|-----------|------------|
| **LIM-01** | Sem OCR — PDFs escaneados (só imagem) não funcionam | Usar Adobe Acrobat ou ferramenta de OCR primeiro |
| **LIM-02** | File System Access API só funciona em Chrome/Edge | Usar Chrome/Edge (Firefox/Safari não suportam) |
| **LIM-03** | Dependência de CDN para libs (pdf-lib, pdf.js) | Em modo offline total, hospedar libs localmente |
| **LIM-04** | Relatório perdido ao fechar a aba | Futura feature: exportar JSON |
| **LIM-05** | Sem testes automatizados | Validação manual via pasta `teste1/` |
| **LIM-06** | Algoritmo v7 com 6 iterações anteriores não versionadas | Histórico implícito no nome da função |

---

## 10. Roadmap

| Versão | Status | Funcionalidade |
|--------|--------|----------------|
| v1 (atual) | ✅ Estável | Split básico de iniciais |
| v1.1 | 🔄 Em teste | Refinamentos do algoritmo v7 |
| v1.2 | 📋 Planejado | Suporte offline completo (libs locais) |
| v2.0 | 💭 Conceitual | Backend opcional (Go + SQLite) com fila de processamento |
| v2.1 | 💭 Conceitual | OCR para PDFs escaneados (Tesseract.js) |
| v3.0 | 💭 Conceitual | Detecção de outras peças (contestação, sentença...) |

---

## 11. Testes

### 11.1 Conjunto de validação
- `processos/` — PDFs integrais padrão (entrada principal)
- `NOVOS/` — PDFs integrais novos
- `erros/` — PDFs com problemas (entrada de casos extremos)
- `procon/` — PDFs de PROCON Estadual

### 11.2 Critérios de aceitação
- **Acurácia mínima:** 85% dos PDFs em `processos/` devem gerar `Inicial_*.pdf` com intervalo correto
- **Zero perda:** nenhum PDF deve ficar sem saída (sucesso ou erro reportado)
- **Reversibilidade:** o arquivo `Docs_*.pdf` concatenado deve ter o mesmo total de páginas do `Integral_*.pdf` original

---

## 12. Conformidade e privacidade

- **LGPD:** nenhum dado pessoal sai do dispositivo do usuário
- **Sem telemetria:** zero analytics, zero requests externos durante operação
- **Sem cookies:** o app não usa cookies ou armazenamento persistente
- **Open source:** código auditável, licença MIT

---

*Especificação mantida em `SPEC.md` — atualizações devem versionar e justificar mudanças neste arquivo.*
