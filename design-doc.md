# governa-rag — Design Document

**Status:** Draft v0.8 — FINAL para implementação  
**Autora:** Luciana Reynaud Ferreira  
**Repositório:** `github.com/[handle]/governa-rag`

---

## 1. Visão e Posicionamento

### 1.1 Problema

Um DPO ou analista de compliance não quer saber o que diz o Art. 46 da LGPD — quer saber se o contrato de processamento de dados que o fornecedor mandou atende o Art. 46. Quer saber se o relatório de incidente que está redigindo configura notificação obrigatória à ANPD. Quer saber se a cláusula de consentimento da nova feature está correta.

O ChatGPT responde perguntas sobre leis. Este produto analisa os documentos do próprio usuário à luz das leis — e cita as duas fontes.

### 1.2 Proposta de Valor

Sistema de análise de conformidade que cruza documentos internos do usuário com corpus normativo de referência (LGPD, PL 2338, EU AI Act, NIST AI RMF), retornando:

- Análise específica sobre o documento submetido, não sobre a lei em abstrato
- Citações duplas: trecho do documento do usuário + norma correspondente
- Identificação de lacunas, cláusulas problemáticas e ausências obrigatórias
- Redação de PII presente no documento do usuário antes de qualquer chamada a API externa
- Avaliação sistemática de faithfulness e attribution via LLM-as-judge

### 1.3 Personas

**Primária:** DPO ou analista de compliance em fintech/healthtech brasileira. Casos de uso:
- Revisão de contratos de processamento de dados com fornecedores
- Validação de RIPDs (Relatório de Impacto à Proteção de Dados)
- Checagem de políticas de privacidade e termos de uso
- Análise de relatórios de incidente antes de notificar a ANPD
- Revisão de cláusulas de consentimento em novas features

**Secundária:** Advogado de privacidade ou escritório de advocacia que atende múltiplos clientes regulados. Caso de uso: triagem rápida de documentos antes de análise jurídica profunda.

### 1.4 Por que não usar ChatGPT ou Claude com web search?

Esta é a pergunta correta e merece resposta direta.

LLMs com acesso à internet em tempo real respondem perguntas sobre leis. Este produto analisa documentos privados à luz de leis — e a diferença não é de grau, é de categoria. Há cinco propriedades estruturais que tornam um sistema RAG privado e auditável irredutível a uma interface de chat, independente do modelo subjacente.

**Confidencialidade do documento analisado.** O caso de uso central envolve contratos com cláusulas estratégicas, RIPDs com dados de titulares, relatórios de incidente com informação regulatoriamente sensível. Esses documentos não podem trafegar para APIs externas sem DPA formal, avaliação de risco e, na prática, consentimento do titular ou autorização legal explícita — condições que na maioria dos contextos empresariais brasileiros não estão satisfeitas. O governa-rag aplica Presidio localmente antes de qualquer embedding e nunca envia o documento do usuário para infraestrutura externa. Isso não é feature: é pré-condição de existência do produto no contexto jurídico-empresarial.

**Corpus autoritativo, estável e rastreável.** Web search retorna o que ranqueia no momento da consulta — um blog jurídico, uma interpretação de escritório de advocacia, um artigo desatualizado anterior a uma emenda podem aparecer antes do texto consolidado oficial. Para análise de conformidade, a fonte importa tanto quanto o conteúdo. O corpus do governa-rag é curado, versionado por `corpus_version`, e cada resposta é rastreável ao `article_ref` e ao hash exato do documento normativo utilizado. Reprodutibilidade de análise — executar a mesma query em datas distintas e obter resposta ancorada na mesma base normativa — é uma propriedade impossível por design em sistemas que dependem de web search.

**Citação dupla como contrato arquitetural.** Um LLM com Browsing pode citar o Art. 46 da LGPD. Não pode simultaneamente ancorar essa citação a um trecho específico de um documento privado que o usuário submeteu — porque o documento privado não está na internet. A arquitetura de dual retriever com `citations[{type: normativo}, {type: user_doc}]` e o guardrail `NO_DOC_ANCHOR` são a materialização técnica desta diferença. Sem citação dupla, o produto regride a chatbot jurídico, categoria já saturada.

**Saída estruturada e integrável.** O produto retorna JSON com schema fixo: `answer`, `citations[]`, `guardrail_status`, `session_id`, `query_id`, `trace_id`. Esse contrato de saída permite integração com workflows de GRC, sistemas de gestão documental, pipelines de auditoria e relatórios exportáveis. Uma interface de chat não é integrável — é terminal. A diferença entre sistema e ferramenta de chat é exatamente esta.

**Cadeia de auditoria verificável.** `session_id → query_id → trace_id` no Langfuse, scores de faithfulness e attribution do judge assíncrono, `corpus_version` na resposta. Em contexto regulatório — especialmente em due diligence, auditorias externas e eventual contencioso —  é necessário demonstrar o que foi analisado, com base em qual versão exata da norma, com qual grau de confiança avaliado e em qual momento. Nenhuma interface de chat oferece esse nível de rastreabilidade por design.

O governa-rag não compete com ChatGPT na dimensão de conhecimento jurídico geral. Compete na dimensão de análise privada, auditável e integrável de documentos confidenciais — um problema que LLMs públicos com web search estruturalmente não resolvem.

---

## 2. Corpus Normativo (Base de Referência)

### 2.1 Documentos Incluídos

**Corpus MVP** (implementado no `manifest.yaml`):

| ID no manifest | Documento | Jurisdição | Idioma | Fetch |
|---|---|---|---|---|
| `lgpd_compilada` | LGPD — Lei 13.709/2018 (texto compilado) | Brasil | PT | HTML (planalto) |
| `pl_2338_2023` | PL 2338/2023 — Marco Legal de IA | Brasil | PT | PDF (senado) |
| `anpd_resolucao_15_incidente` | Resolução CD/ANPD nº 15/2024 — Incidentes | Brasil | PT | PDF (gov.br/anpd) |
| `eu_ai_act_2024` | EU AI Act — Regulation (EU) 2024/1689 | EU | EN | PDF (eur-lex) |
| `nist_ai_rmf_1_0` | NIST AI RMF 1.0 | EUA | EN | PDF (nvlpubs.nist.gov) |

**Corpus v1** (adicionado quando MVP estiver estável):

| Documento | Justificativa de inclusão | Bloqueador |
|---|---|---|
| ANPD — Guia de Boas Práticas para RIPD | Necessário para análise de RIPDs, caso de uso primário | URL pública disponível |
| ANPD — Guia de Segurança da Informação | Complementa análise de contratos e políticas de segurança | URL pública disponível |
| Resoluções BCB 3040 e 4658 | Relevante para clientes fintech com obrigações BACEN | PDFs disponíveis em bcb.gov.br |
| ISO/IEC 42001:2023 | Framework de gestão de IA; versão paga — requer aquisição ou substituição por artefato derivado público | **Bloqueado:** norma publicada é paga; draft público retirado em 2024 |

> **Decisão de escopo:** ISO/IEC 42001 permanece fora do MVP e de v1 até que seja identificada fonte pública acessível (comentário técnico ABNT, summary oficial, ou aquisição da norma). Não incluir fonte inacessível no manifest evita falha silenciosa no ingestor.

### 2.2 Pipeline de Ingestão (Corpus Normativo)

```
PDF (fonte primária)
  → parsing (pdfplumber + pymupdf → fallback unstructured hi_res)
  → chunking hierárquico por estrutura legal (artigo > inciso > alínea)
  → fallback sliding window 512 tokens, overlap 128
  → embedding multilingual-e5-large
  → upsert Qdrant local (Docker) com payload estruturado
```

Metadados obrigatórios: `source_doc`, `article_ref`, `jurisdiction`, `language`, `ingestion_date`, `chunk_id`, `corpus_version`, `content_hash`, `active`.

Versionamento idempotente: chunks com mesmo `(source_doc, article_ref, content_hash)` são ignorados. Versões novas deprecam via `active: false` — histórico auditável preservado.

---

## 3. Documentos do Usuário (Contexto de Sessão)

### 3.1 Natureza

Documentos submetidos pelo usuário são **efêmeros e isolados por sessão**. Não são indexados no Qdrant permanentemente — são parseados, chunkados e mantidos em memória (ou em store temporário com TTL de sessão). Isso tem três motivações:

- Dados sensíveis do usuário (CPFs, dados de titulares, informações confidenciais) não devem persistir em infraestrutura externa
- Cada análise é contextualmente independente — não há valor em cruzar o contrato do cliente A com o contexto do cliente B
- Simplifica LGPD compliance do próprio produto: dados de sessão com TTL é muito mais fácil de justificar do que indexação persistente

### 3.2 Formatos Suportados (MVP)

- PDF (contratos, RIPDs, políticas, relatórios de incidente)
- Texto plano (.txt, .md)

### 3.3 Pipeline de Processamento do Documento do Usuário

```
Upload (PDF / texto)
  → parsing (pdfplumber → fallback unstructured hi_res)
  → Presidio: redação de PII antes de qualquer processamento externo
  → chunking semântico (512 tokens, overlap 128)
  → embedding multilingual-e5-large (em memória ou store temporário)
  → disponível para retrieval dual na sessão
```

**Presidio aqui é obrigatório e justificado:** contratos têm CPFs de representantes legais, relatórios de incidente têm dados de titulares afetados, RIPDs descrevem categorias de dados com exemplos. A redação acontece antes do embedding — o vetor gerado não carrega o dado em claro, e a API de embedding externa nunca recebe o PII.

---

## 4. Arquitetura

### 4.1 Visão Geral

```
Client (HTTP) — envia {document, question, jurisdiction[]}
    │
    ▼
┌─────────────────────────────────┐
│           API Gateway           │  FastAPI · rate limit · auth · cost tracking
│         (gateway/)              │  OTel SDK → OTLP exporter
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────┐
│                   RAG Pipeline                      │
│                    (rag/)                           │
│                                                     │
│  ┌─────────────────┐    ┌──────────────────────┐   │
│  │ Doc Processor   │    │  Presidio (doc input) │   │
│  │ parse + chunk   │───▶│  PII redaction antes  │   │
│  │ user document   │    │  de embedding         │   │
│  └────────┬────────┘    └──────────┬────────────┘   │
│           │                        │                 │
│           ▼                        ▼                 │
│  ┌─────────────────┐    ┌──────────────────────┐   │
│  │  Session Store  │    │  Embedder            │   │
│  │  (doc chunks    │◀───│  multilingual-e5     │   │
│  │   + vectors)    │    └──────────────────────┘   │
│  └────────┬────────┘                                │
│           │                                         │
│           ▼                                         │
│  ┌────────────────────────────────────────────┐    │
│  │           Dual Retriever                   │    │
│  │                                            │    │
│  │  query → Qdrant local  (corpus normativo) │    │
│  │  query → Session Store  (doc do usuário)   │    │
│  │                                            │    │
│  │  merge + rerank por relevância             │    │
│  └────────────────────┬───────────────────────┘    │
│                       │                             │
│                       ▼                             │
│  ┌────────────────────────────────────────────┐    │
│  │         Prompt Builder                     │    │
│  │  [trecho do doc usuário] + [norma citada]  │    │
│  └────────────────────┬───────────────────────┘    │
│                       │                             │
└───────────────────────┼─────────────────────────────┘
                        │
                        ▼
              ┌───────────────────┐
              │    LLM Router     │  LiteLLM — GPT-4o / Claude Haiku
              └────────┬──────────┘
                       │
                       ▼
              ┌───────────────────┐
              │ Presidio (output) │  defesa em profundidade
              └────────┬──────────┘
                       │
                       ▼
              Response + CitationPairs[]

              ┌───────────────────────┐
              │  Judge Worker (async) │
              │  faithfulness +       │
              │  attribution (dual)   │
              └───────────────────────┘

              ┌───────────────────────┐
              │      Dashboard        │
              └───────────────────────┘
```

### 4.2 Módulos

| Módulo | Path | Responsabilidade |
|---|---|---|
| `gateway/` | `src/gateway/` | Auth, rate limit, cost tracking, OTel spans |
| `rag/` | `src/rag/` | Dual retrieval, prompt assembly, generation |
| `doc_processor/` | `src/doc_processor/` | Parse, chunk e PII redaction de documentos do usuário |
| `ingestor/` | `src/ingestor/` | Ingestão do corpus normativo no Qdrant |
| `session/` | `src/session/` | Store temporário de vetores de sessão com TTL |
| `judge/` | `src/judge/` | Worker assíncrono de avaliação de qualidade |
| `pii/` | `src/pii/` | Presidio com PatternRecognizer custom BR |
| `dashboard/` | `src/dashboard/` | Streamlit consumindo Langfuse API |
| `corpus/` | `corpus/` | Manifesto e scripts de download |

---

## 5. Componentes Técnicos

### 5.1 Vector Store — Qdrant Local (Docker)

**Motivação para não usar Qdrant Cloud no MVP:** o free tier suspende automaticamente após 1 semana de inatividade e deleta o cluster após 4 semanas sem reativação. Isso inviabiliza demo confiável — uma demonstração agendada com 10 dias de antecedência quebraria silenciosamente. Qdrant local no Docker Compose elimina essa dependência, mantém todos os dados em ambiente controlado e tem custo zero.

**Para v1/prod:** Qdrant Cloud pago (sem política de suspensão) ou Qdrant self-hosted em VPS. A migração é trivial — apenas a URL de conexão muda.

**Docker Compose:**
```yaml
qdrant:
  image: qdrant/qdrant:latest
  ports:
    - "6333:6333"
    - "6334:6334"   # gRPC
  volumes:
    - qdrant_storage:/qdrant/storage
  environment:
    QDRANT__SERVICE__GRPC_PORT: "6334"

volumes:
  qdrant_storage:
```

**Conexão:**
```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams

client = QdrantClient(host="qdrant", port=6333)  # "localhost" fora do Docker

client.create_collection(
    collection_name="governa_normativo",
    vectors_config=VectorParams(size=1024, distance=Distance.COSINE),
)
```

**Payload por ponto:**
```json
{
  "source_doc": "lgpd",
  "article_ref": "Art. 46",
  "jurisdiction": "BR",
  "language": "pt",
  "corpus_version": "1.0.0",
  "content_hash": "sha256:...",
  "active": true,
  "text": "..."
}
```

Filtro obrigatório: `active: true` + `jurisdiction` quando aplicável.

### 5.2 Session Store (Documentos do Usuário)

Vetores de sessão são mantidos em memória (MVP) ou Redis com TTL configurável (v1). Estrutura:

```python
{
  "session_id": "uuid4",          # gerado no upload, retornado ao cliente
  "chunks": [
    {
      "chunk_id": "doc-p1-c3",
      "text": "...",              # texto já redigido pelo Presidio
      "vector": [...],
      "page": 1,
      "char_offset": 342,
    }
  ],
  "doc_metadata": {
    "filename": "contrato_processamento_fornecedor_x.pdf",
    "page_count": 12,
    "ingested_at": "2025-01-01T10:00:00Z",
    "pii_entities_found": ["BR_CPF", "PERSON"],   # tipos, nunca valores
  },
  "created_at": "2025-01-01T10:00:00Z",
  "last_accessed_at": "2025-01-01T10:05:00Z",
  "expires_at": "2025-01-01T11:05:00Z"    # sliding TTL: recalculado a cada query
}
```

**Lifecycle da sessão (MVP — in-memory):**

- **TTL:** sliding, 1 hora após última atividade. Cada query bem-sucedida renova `last_accessed_at` e `expires_at`.
- **Expiração:** verificada lazily no momento do acesso. Uma query com `session_id` expirado retorna `SESSION_EXPIRED` — o cliente deve re-enviar o documento.
- **Reinício de processo:** sessões em memória são perdidas. Comportamento esperado e documentado — o cliente recebe `SESSION_NOT_FOUND` e deve re-enviar. Não há tentativa de recuperação silenciosa.
- **Isolamento (MVP):** o MVP é explicitamente single-tenant — uma única chave de API, um único operador. Nesse contexto, `session_id` como mecanismo de acesso é suficiente. Não há endpoint de listagem de sessões. A propriedade de isolamento por tenant (`tenant_id + session_id` como chave composta no gateway) é uma adição de v1, quando múltiplos clientes passarem a compartilhar a infraestrutura. Não existe essa propriedade no MVP — e isso é uma decisão consciente, não uma lacuna.
- **Worker stickiness (MVP):** processo único, sem necessidade de afinidade. Em v1 com múltiplos workers, Redis resolve o problema — sessão passa a ser acessível por qualquer worker.

**Em memória para MVP** — elimina dependência de Redis no Docker Compose inicial. Redis entra em v1 para suportar múltiplos workers.

### 5.3 Embeddings — Local via sentence-transformers

Todos os embeddings — corpus normativo e documentos do usuário — rodam localmente via `sentence-transformers`. Isso é obrigatório para o documento do usuário (dados confidenciais não devem trafegar para APIs externas mesmo após redação de PII) e consistente para o corpus normativo (mesmo modelo, mesma escala de scores, zero custo por token).

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("intfloat/multilingual-e5-large")
# roda 100% local, sem chamada de API externa
# 1024 dimensões, suporte nativo PT + EN

def embed(texts: list[str]) -> list[list[float]]:
    return model.encode(
        texts,
        normalize_embeddings=True,   # obrigatório para cosine similarity correta
        batch_size=32,
    ).tolist()
```

**Performance:** em CPU, ~4-8s para um contrato de 30 páginas (~300 chunks). Aceitável para MVP — o processamento ocorre uma vez no upload, não por query. Para v1, adicionar cache de embeddings por `content_hash` e avaliar `multilingual-e5-small` como fallback para documentos grandes.

**Footprint e cold start:** `multilingual-e5-large` requer ~2.5GB de RAM para carregar o modelo e os pesos. Cold start (primeira inferência após carregamento) leva 3-8s em CPU. O modelo deve ser carregado uma vez na inicialização do processo (`lifespan` do FastAPI), não por request. Requisito mínimo recomendado para o contêiner: 4GB RAM. Em máquinas com menos de 3GB disponíveis, usar `multilingual-e5-small` (512 dim, ~500MB) com redução de dimensionalidade no Qdrant.

**GPU:** se disponível, `SentenceTransformer` usa automaticamente via `device="cuda"`. Sem alteração de código.

### 5.4 Dual Retriever

O ponto central do produto. Para cada query, recupera de duas fontes e mescla via RRF — mesmo no MVP dense-only, RRF é preferível a merge por score bruto porque cosine similarity do Qdrant e scores do session store podem ter distribuições diferentes mesmo usando o mesmo modelo.

```python
async def retrieve(
    query: str,
    session_id: str,
    jurisdictions: list[str],
    k_normativo: int = 4,
    k_doc: int = 3,
) -> list[RetrievedChunk]:

    query_vector = embed([f"query: {query}"])[0]  # prefixo e5-large para queries

    # Fonte 1: corpus normativo no Qdrant local
    normativo_hits = await qdrant_search(
        vector=query_vector,
        filter={"active": True, "jurisdiction": {"$in": jurisdictions}},
        k=k_normativo,
    )

    # Fonte 2: documento do usuário na session store
    doc_hits = await session_search(
        vector=query_vector,
        session_id=session_id,
        k=k_doc,
    )

    # RRF: desacopla merge de escala absoluta dos scores
    return reciprocal_rank_fusion(normativo_hits, doc_hits, k=60)
```

Cada chunk retornado carrega `source_type: "normativo" | "user_doc"` — obrigatório para citação dupla no prompt e para o judge de attribution.

**Retrieval dense-only no MVP.** Hybrid search com Qdrant sparse vectors (nativo, sem BM25 externo) entra em v1.

### 5.5 Prompt Builder

Template versionado com dois blocos de contexto explicitamente separados:

```jinja2
Você é um assistente especializado em conformidade regulatória.

## Trechos relevantes do documento analisado
{% for chunk in user_doc_chunks %}
[DOC · p.{{ chunk.page }}] {{ chunk.text }}
{% endfor %}

## Normas aplicáveis
{% for chunk in normativo_chunks %}
[{{ chunk.source_doc | upper }} · {{ chunk.article_ref }}] {{ chunk.text }}
{% endfor %}

## Pergunta
{{ question }}

## Instrução
Responda exclusivamente com base nos trechos acima.
Para cada afirmação, cite o trecho do documento analisado E a norma correspondente.
Se o documento analisado não contiver informação suficiente para responder, diga explicitamente.
Não extrapole além do contexto fornecido.
```

**Saída estruturada obrigatória:** a geração ocorre em modo structured output (JSON schema via `response_format` do LiteLLM/OpenAI), não em texto livre. Os guardrails operam sobre o objeto estruturado retornado, não sobre parsing textual pós-hoc. Isso elimina a fragilidade de heurísticas de parsing e torna os guardrails determinísticos.

Schema de saída esperado do LLM:
```json
{
  "answer": "string",
  "citations": [
    {
      "type": "user_doc | normativo",
      "ref": "p.3 | Art. 46",
      "source_doc": "lgpd",
      "excerpt": "string"
    }
  ]
}
```

**Guardrail pós-geração** opera sobre o objeto acima:
- `NO_GROUNDED_ANSWER`: `citations` está vazio ou nulo
- `NO_DOC_ANCHOR`: `citations` não contém nenhum item com `type: "user_doc"`

Se o LLM retornar JSON inválido ou não aderente ao schema, a resposta é rejeitada com `GENERATION_SCHEMA_ERROR` antes de qualquer guardrail — não há fallback para parsing textual.

### 5.6 Presidio — PatternRecognizer Custom BR

```python
from presidio_analyzer import AnalyzerEngine, PatternRecognizer, Pattern
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

cpf_recognizer = PatternRecognizer(
    supported_entity="BR_CPF",
    patterns=[
        Pattern(name="cpf_formatted", regex=r"\b\d{3}\.\d{3}\.\d{3}-\d{2}\b", score=0.85),
        Pattern(name="cpf_unformatted", regex=r"\b\d{11}\b", score=0.5),
    ],
)

cnpj_recognizer = PatternRecognizer(
    supported_entity="BR_CNPJ",
    patterns=[
        Pattern(name="cnpj_formatted", regex=r"\b\d{2}\.\d{3}\.\d{3}/\d{4}-\d{2}\b", score=0.9),
    ],
)

analyzer = AnalyzerEngine()
analyzer.registry.add_recognizer(cpf_recognizer)
analyzer.registry.add_recognizer(cnpj_recognizer)

anonymizer = AnonymizerEngine()
ENTITIES = ["PERSON", "EMAIL_ADDRESS", "PHONE_NUMBER", "BR_CPF", "BR_CNPJ", "LOCATION"]

def redact(text: str, language: str = "pt") -> tuple[str, list[str]]:
    results = analyzer.analyze(text=text, language=language, entities=ENTITIES)
    redacted = anonymizer.anonymize(
        text=text,
        analyzer_results=results,
        operators={"DEFAULT": OperatorConfig("replace", {"new_value": "[REDACTED·<entity_type>]"})},
    )
    entity_types = list({r.entity_type for r in results})
    return redacted.text, entity_types
```

**Dois pontos de aplicação:**

Ponto 1 — documento do usuário, antes do embedding. Motivação real: contratos têm CPFs de representantes, RIPDs descrevem dados de titulares com exemplos, relatórios de incidente identificam afetados. O embedding externo nunca recebe PII.

Ponto 2 — output do LLM, antes de retornar ao cliente. Defesa em profundidade contra vazamento de PII que eventualmente sobreviveu ao ponto 1 (ex: PII em chunk normativo mal parseado, improvável mas possível).

A query do usuário não passa por Presidio por padrão. A motivação é que queries de conformidade estruturalmente não deveriam conter PII — a pergunta típica é "esta cláusula atende o Art. 46?" e não "verifique se o CPF 123.456.789-00 está exposto". Porém, isso é uma premissa de produto, não uma garantia técnica. Usuários podem colar trechos textuais na pergunta ou formular queries como "esta cláusula sobre João da Silva está correta?". Esse failure mode existe e é reconhecido. Em v1, adicionar scrubbing condicional da query (acionado quando o detector de alta confiança do Presidio identificar entidades com score > 0.85) é o caminho correto — sem prejudicar queries legítimas com redação desnecessária.

### 5.7 Langfuse

```
trace: session_id + query_id
  ├── span: gateway.auth
  ├── span: doc_processor.parse
  │     metadata: {filename, page_count, parse_strategy}
  ├── span: pii.redaction_doc
  │     metadata: {entities_found: count, types: [...]}
  ├── span: rag.dual_retrieval
  │     metadata: {k_normativo, k_doc, normativo_hits, doc_hits, scores}
  ├── span: rag.prompt_assembly
  │     metadata: {template_version, context_tokens, user_doc_chunks: n, normativo_chunks: n}
  ├── span: rag.generation
  │     metadata: {model, input_tokens, output_tokens, latency_ms, cost_usd}
  ├── span: pii.redaction_output
  │     metadata: {entities_found: count, types: [...]}
  └── score: judge.faithfulness    (assíncrono)
      score: judge.attribution_doc (assíncrono)
      score: judge.attribution_norm (assíncrono)
```

### 5.8 Judge Worker

Para cada trace sem score, avalia três dimensões:

- **Faithfulness:** a resposta é integralmente suportada pelos chunks fornecidos (user_doc + normativo)?
- **Attribution doc:** cada afirmação sobre o documento do usuário cita o trecho correto?
- **Attribution norm:** cada afirmação normativa cita a norma e artigo corretos?

Prompts com few-shot (2 positivos + 1 negativo por dimensão). Modelo: GPT-4o-mini. Threshold de alerta: faithfulness < 0.80, attribution_doc < 0.75, attribution_norm < 0.75.

**Limitações conhecidas e documentadas:**

Os thresholds acima são iniciais e arbitrários — não foram calibrados contra um conjunto de referência. A calibração correta requer: (1) golden set anotado manualmente com scores humanos de referência, (2) correlação entre score do judge e julgamento humano, (3) ajuste dos thresholds para minimizar falsos negativos em contexto regulatório (prefere-se alerta excessivo a silêncio em falha real). Calibração entra no roadmap de v1.

O judge opera no nível de resposta (response-level), não no nível de afirmação (claim-level). Uma resposta pode ter faithfulness 0.90 e ainda assim conter uma afirmação crítica sem suporte nos chunks — a afirmação problemática é diluída pelo score agregado. Em contexto regulatório, esse é o failure mode mais perigoso. A evolução natural é decompor a resposta em proposições verificáveis individualmente (claim decomposition), o que requer um LLM mais capaz que GPT-4o-mini para extração de claims. Isso fica explicitamente fora do escopo do MVP e da v1 básica.

### 5.9 Gateway

- Auth: API key via `X-API-Key`
- Rate limiting: por chave, por tier
- Cost tracking: custo real via LiteLLM callbacks, acumulado por tenant
- OTel: `FastAPIInstrumentor`, spans via OTLP

### 5.10 Dashboard

- **Qualidade:** scores de faithfulness e attribution por período, modelo, tipo de documento
- **Custo:** custo real por tenant, modelo, dia; projeção mensal
- **Sessões:** volume de documentos processados, tipos de PII encontrados (contagem agregada, nunca valores), distribuição de erros (NO_GROUNDED_ANSWER, NO_DOC_ANCHOR)

---

## 6. Modelos Internos

Contratos entre módulos. Ausência desses tipos na implementação é a principal fonte de ambiguidade entre componentes.

```python
@dataclass
class RetrievedChunk:
    chunk_id: str
    text: str                        # já redigido (user_doc) ou original (normativo)
    vector: list[float]
    score: float                     # cosine similarity pré-RRF
    rrf_score: float                 # score pós-fusão
    source_type: Literal["normativo", "user_doc"]
    # campos normativo
    source_doc: str | None           # ex: "lgpd"
    article_ref: str | None          # ex: "Art. 46"
    jurisdiction: str | None         # ex: "BR"
    # campos user_doc
    page: int | None
    char_offset: int | None

@dataclass
class Citation:
    type: Literal["normativo", "user_doc"]
    ref: str                         # "Art. 46" ou "p.3"
    source_doc: str | None
    excerpt: str

@dataclass
class AnalyzeResponse:
    answer: str
    citations: list[Citation]
    guardrail_status: Literal["ok", "NO_GROUNDED_ANSWER", "NO_DOC_ANCHOR", "GENERATION_SCHEMA_ERROR"]
    session_id: str
    query_id: str
    trace_id: str                    # Langfuse trace ID, exposto para debugging

@dataclass
class JudgeResult:
    query_id: str
    faithfulness: float              # [0.0, 1.0]
    faithfulness_comment: str
    attribution_doc: float
    attribution_doc_comment: str
    attribution_norm: float
    attribution_norm_comment: str
    evaluated_at: str
```

**Relação entre identificadores:**

- `session_id`: gerado no primeiro upload, vinculado ao documento do usuário. Persiste enquanto a sessão estiver ativa. Retornado ao cliente para reuso.
- `query_id`: gerado por request de análise. `uuid4`. Vincula o request ao trace do Langfuse e ao `JudgeResult` assíncrono.
- `trace_id`: gerado pelo Langfuse SDK ao abrir a trace. Corresponde 1:1 ao `query_id` como `external_id` no Langfuse. Exposto na resposta da API para debugging e correlação manual em auditoria.

## 7. API Shape

```
POST /analyze
Content-Type: multipart/form-data

Fields:
  document: File (PDF ou texto) — obrigatório se session_id ausente
  question: string
  jurisdictions: string[] (ex: ["BR", "EU"])
  session_id: string (opcional — reutiliza doc já processado na sessão)

Errors:
  SESSION_EXPIRED    → 410 Gone
  SESSION_NOT_FOUND  → 404 Not Found
  DOCUMENT_REQUIRED  → 422 Unprocessable (session_id ausente + sem document)

Response 200:
{
  "answer": "string",
  "citations": [
    {
      "type": "user_doc",
      "ref": "p.3",
      "source_doc": null,
      "excerpt": "O operador se compromete a..."
    },
    {
      "type": "normativo",
      "ref": "Art. 46",
      "source_doc": "lgpd",
      "excerpt": "Os agentes de tratamento devem adotar..."
    }
  ],
  "guardrail_status": "ok" | "NO_GROUNDED_ANSWER" | "NO_DOC_ANCHOR" | "GENERATION_SCHEMA_ERROR",
  "session_id": "uuid",
  "query_id": "uuid",
  "trace_id": "uuid"
}
```

`session_id` retornado permite múltiplas perguntas sobre o mesmo documento sem re-upload e re-processamento.

---


## 8. Stack Tecnológica

| Camada | MVP | v1/Prod |
|---|---|---|
| Vector store (normativo) | Qdrant local (Docker) | Qdrant Cloud pago / self-hosted em VPS |
| Session store (user doc) | In-memory | Redis com TTL |
| Embeddings | multilingual-e5-large local (sentence-transformers) | idem + cache por content_hash |
| Retrieval | Dense-only dual + RRF | Hybrid sparse+dense RRF (Qdrant native) |
| LLM routing | LiteLLM | idem |
| Tracing | Langfuse local (Docker Compose) | Langfuse self-hosted em VPS / Langfuse Cloud |
| OTel | opentelemetry-sdk + OTLP | idem |
| PII | Presidio + PatternRecognizer BR | idem |
| Judge | GPT-4o-mini (3 dimensões) | idem |
| Dashboard | Streamlit | React (v2) |
| PDF parsing | pdfplumber + pymupdf + unstructured fallback | idem |
| Infra local | Docker Compose | — |

---

## 9. Estrutura de Repositório

```
governa-rag/
├── corpus/
│   ├── manifest.yaml
│   ├── download.py               # fetch + idempotência por content_hash
│   ├── watch.py                  # detecção de mudança + alerta (v1)
│   └── raw/                      # PDFs e HTMLs baixados; não versionado no git
├── src/
│   ├── gateway/
│   │   ├── main.py
│   │   └── middleware/
│   │       ├── auth.py
│   │       ├── rate_limit.py
│   │       └── cost_tracker.py
│   ├── rag/
│   │   ├── dual_retriever.py
│   │   ├── prompt_builder.py
│   │   ├── generator.py
│   │   └── prompts/
│   │       └── v1_analyze.jinja2
│   ├── doc_processor/
│   │   ├── parser.py
│   │   └── chunker.py
│   ├── ingestor/
│   │   ├── parser.py
│   │   ├── chunker.py
│   │   ├── embedder.py
│   │   └── pipeline.py
│   ├── session/
│   │   └── store.py              # in-memory MVP; Redis v1
│   ├── judge/
│   │   ├── worker.py
│   │   └── prompts/
│   │       ├── faithfulness.jinja2
│   │       ├── attribution_doc.jinja2
│   │       └── attribution_norm.jinja2
│   ├── pii/
│   │   └── redactor.py
│   └── dashboard/
│       └── app.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── eval/
│       └── golden_set.jsonl      # {document_excerpt, question, expected_citations, forbidden}
├── docker-compose.yml             # Qdrant + Langfuse + app (Redis entra em v1)
├── pyproject.toml
├── .env.example
└── README.md
```

---

## 10. Fluxo de Dados — Análise End-to-End

```
1.  Cliente envia POST /analyze {document, question, jurisdictions}
2.  Gateway valida API key → rate limit → abre span OTel
3.  doc_processor parseia o documento do usuário
4.  Presidio redige PII do documento → loga {count, types} (nunca valores)
5.  Embedder gera vetores dos chunks redigidos → session store (TTL)
6.  Dual retriever executa em paralelo:
      → Qdrant: top-4 chunks normativos filtrados por jurisdiction + active
      → Session store: top-3 chunks do documento do usuário
7.  PromptBuilder monta prompt com dois blocos contextuais separados
8.  Guardrail pré-geração: contexto mínimo em ambas as fontes?
9.  Generator chama LLM via LiteLLM → recebe resposta
10. Guardrail pós-geração:
      → sem par de citação (user_doc + normativo) → NO_GROUNDED_ANSWER
      → citação apenas normativa, sem ancoragem no doc → NO_DOC_ANCHOR
11. Presidio redige PII do output (defesa em profundidade)
12. Resposta retorna ao cliente com citations[] estruturadas + session_id
13. LiteLLM callback registra custo real → Gateway fecha span OTel
14. [Async] Judge Worker avalia faithfulness + attribution_doc + attribution_norm
      → grava 3 scores no Langfuse
      → alerta se qualquer score < threshold
15. Dashboard consome Langfuse API → exibe painéis
```

---

## 11. Avaliação e Golden Set

`tests/eval/golden_set.jsonl` — cada entrada:

```json
{
  "document_excerpt": "O fornecedor processará dados pessoais...",
  "question": "Esta cláusula atende os requisitos do Art. 46 da LGPD?",
  "expected_citation_sources": ["lgpd:art46", "user_doc:p2"],
  "expected_answer_contains": ["medidas técnicas", "organizacionais"],
  "forbidden": ["sem riscos identificados"]
}
```

CI a cada push em `main`: retrieval recall@dual, faithfulness médio, attribution_doc médio, attribution_norm médio, latência p50/p95, custo médio por análise.

---

## 12. Roadmap

### MVP (portfólio + demo)

- [ ] Ingestor do corpus normativo (5 documentos do manifest: LGPD, PL 2338, ANPD Resolução 15, EU AI Act, NIST AI RMF)
- [ ] doc_processor: parse + chunking de documentos do usuário
- [ ] Presidio no documento do usuário (PatternRecognizer BR custom)
- [ ] Session store in-memory com TTL
- [ ] Dual retriever (Qdrant local + session store)
- [ ] Prompt template com dois blocos contextuais
- [ ] Guardrails: NO_GROUNDED_ANSWER + NO_DOC_ANCHOR + GENERATION_SCHEMA_ERROR
- [ ] Langfuse instrumentado end-to-end
- [ ] Judge com 3 dimensões (few-shot)
- [ ] API shape com session_id e citations[]
- [ ] Docker Compose (Qdrant + Langfuse + app)
- [ ] README com exemplo de análise real + métricas do golden set

### v1 (produto)

- [ ] Redis para session store (multi-worker)
- [ ] Hybrid search com Qdrant sparse vectors (RRF nativo)
- [ ] Gateway completo com auth + rate limit + cost tracking real
- [ ] OTel exportando para backend configurável
- [ ] Dashboard Streamlit com os três painéis
- [ ] Multi-tenancy (tenant_id no payload Qdrant + filtro obrigatório)
- [ ] Corpus expandido: ANPD Guia RIPD + Guia Segurança + Resoluções BCB (manifest v1)
- [ ] `corpus/watch.py` — detecção de mudança por content_hash + alerta + re-ingestão semi-automática após aprovação humana
- [ ] Scrubbing condicional da query via Presidio (score > 0.85)
- [ ] Golden set com 50+ entradas curadas
- [ ] CI com threshold de regressão por métrica (recall@dual, faithfulness, latência)

### v2 (SaaS)

- [ ] UI web (React) — upload de documento + interface de perguntas
- [ ] Planos por tier (volume de análises / páginas processadas)
- [ ] Relatório exportável por análise (PDF com citações duplas)
- [ ] Corpus customizado por tenant (documentos internos da empresa como referência adicional)
- [ ] Migração para pgvector/Supabase se multi-tenancy exigir RLS por schema

---

## 13. Decisões de Design e Trade-offs

**Qdrant local vs Qdrant Cloud free tier:** o free tier suspende automaticamente após 1 semana de inatividade e deleta o cluster após 4 semanas. Uma demo agendada com 10 dias de antecedência quebraria silenciosamente. Qdrant local no Docker Compose é mais confiável, mantém dados em ambiente controlado e tem custo zero. A migração para Qdrant Cloud pago em v1 é trivial — apenas a URL de conexão muda.

**Embeddings locais (sentence-transformers) para ambas as fontes:** obrigatório para documentos do usuário — dados confidenciais não devem trafegar para APIs externas mesmo após redação de PII. Estender a mesma decisão para o corpus normativo garante escala de scores consistente entre as duas fontes e elimina custo por token de embedding. O modelo `multilingual-e5-large` roda 100% local sem degradação de qualidade relevante frente a APIs externas para este caso de uso.

**RRF no merge dual, mesmo no MVP dense-only:** cosine similarity do Qdrant e scores do session store podem ter distribuições diferentes dependendo da densidade de cada corpus. RRF desacopla o merge de escala absoluta de scores e é robusto a essa variação sem calibração manual — custo de implementação é zero dado que o código já estava no doc anterior.

**Documentos do usuário efêmeros vs indexados permanentemente:** dados de contratos e RIPDs têm alto potencial de conter PII e informação confidencial. Indexação permanente exigiria isolamento rigoroso por tenant, DPA com fornecedores e gestão de retenção. Session store com TTL é muito mais simples de justificar sob LGPD — os dados existem apenas pelo tempo necessário para responder as perguntas da sessão.

**Dual retriever vs RAG convencional:** o diferencial do produto é a citação dupla — ancoragem no documento do usuário E na norma. Sem o dual retriever, o produto regride a um chatbot de perguntas sobre leis, que o ChatGPT já faz. O dual retriever é a decisão que define o produto.

**Presidio no documento do usuário, não na query (por padrão):** contratos e RIPDs contêm PII por natureza; queries de conformidade estruturalmente não deveriam. A exclusão da query é uma premissa de produto, não uma garantia técnica — o failure mode de usuários colando PII na pergunta existe e está documentado. Em v1, scrubbing condicional da query (threshold de confiança > 0.85) resolve sem penalizar queries legítimas.

**NO_DOC_ANCHOR como guardrail específico:** resposta que cita apenas a norma sem ancorar no documento do usuário é tecnicamente faithful, mas falha o propósito do produto. O usuário quer saber sobre o seu documento, não sobre a lei em abstrato.

**In-memory para session store no MVP:** elimina Redis do Docker Compose inicial. Limitação: uma única instância do processo. Redis entra em v1 quando múltiplos workers forem necessários.

**Dense-only no MVP:** implementar hybrid search antes de ter o pipeline dual funcionando é a ordem errada. A limitação é documentada. Hybrid com Qdrant sparse vectors (nativo, sem BM25 externo) entra em v1.
