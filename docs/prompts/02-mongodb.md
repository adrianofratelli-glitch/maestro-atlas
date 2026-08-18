# Torre — Atlas Control Plane — MongoDB e Atlas Admin API

> Segundo dos três prompts. Aqui o MongoDB aparece em três papéis: o alvo da observação (via Admin API), a memória do assistente, e o destino da única ação direta que a ferramenta executa.

---

## Os três papéis

| Papel | Onde |
|---|---|
| **Alvo observado** | a frota inteira, lida pela Atlas Admin API v2 — clusters, medições, alertas, Performance Advisor, Query Profiler, fatura |
| **Memória do assistente** | uma coleção de conversas no Atlas, via `pymongo` (`MONGODB_URI` opcional) |
| **Ação direta** | criação de índice no cluster observado, a partir da recomendação do Performance Advisor |

## O que vem da Admin API

Leitura de: lista de projetos e clusters (tier, região, versão, storage, auto-scaling), medições de processo e de disco, séries de 24h, alertas abertos, sugestões do Performance Advisor, slow queries do Query Profiler e a fatura da organização.

Três regras do cliente que valem repetir aqui porque afetam o número que aparece na tela:

- **Cache com TTL nas chamadas caras** — resolver o primário custa uma sequência de chamadas.
- **Métrica vem do primário**, resolvido explicitamente.
- **Ponto nulo é ponto nulo** — pega o último valor não-nulo em vez de assumir zero. Zero e "não reportou" são coisas diferentes, e confundir as duas produz um scale down lindo e errado.

## A memória do chat — `chat_memory.py`

Um documento por conversa: `title`, `cluster`, `messages[]`, `updated_at`.

`init_db()` é **idempotente** — checa `list_indexes()` antes de criar cada um:

```python
# busca textual sobre título e conteúdo das mensagens
coll.create_index([("title", TEXT), ("messages.content", TEXT)],
                  name="text_search", default_language="portuguese")

# listagem de conversas recentes
coll.create_index([("updated_at", DESCENDING)], name="updated_at_desc")

# filtro por contexto de cluster
coll.create_index([("cluster", DESCENDING)], name="cluster_idx")

# retenção
coll.create_index([("updated_at", 1)], name="updated_at_ttl",
                  expireAfterSeconds=CHAT_RETENTION_DAYS * 86400)
```

O índice de texto usa `default_language="portuguese"` porque as conversas são em português — stemming errado degrada a busca em silêncio.

E a busca de conversa é `$text` com `{"$meta": "textScore"}` ordenando o resultado. **Full-text sem Elasticsearch** — é um argumento pequeno, mas coerente com o resto do portfólio: mais um componente que não precisou entrar na arquitetura.

O TTL em `updated_at` existe pelo motivo de sempre: histórico de demo que fica pra sempre vira custo pra sempre.

## As guardas — as duas superfícies onde entrada de usuário toca o banco

### 1. `conversation_id`

Convertido por um `_oid()` que valida antes de instanciar `ObjectId`. Id de string livre indo direto pro driver é erro esperando acontecer.

### 2. Criação de índice

Namespace e chaves vêm da UI (a partir da sugestão do Performance Advisor) e **passam por gramática explícita** antes de chegar no driver:

```python
_NAMESPACE_RE = re.compile(r"^[A-Za-z0-9_-]{1,63}\.[A-Za-z0-9_-]{1,120}$")
_PROTECTED_DATABASES = {"admin", "config", "local"}

_INDEX_FIELD_RE  = re.compile(r"^(?:[A-Za-z_][A-Za-z0-9_-]*)(?:\.[A-Za-z_][A-Za-z0-9_-]*)*$")
_INDEX_DIRECTIONS = {1, -1, "1", "-1", "hashed", "2dsphere", "text"}
```

Regras cobradas:

- namespace tem que ser `database.collection` e **não pode ser um dos databases protegidos** (`admin`, `config`, `local`);
- cada chave de índice tem **exatamente um** campo;
- o nome do campo casa a gramática de caminho (com suporte a subdocumento por ponto);
- a direção/tipo está na lista fechada.

E o corpo aceita no máximo 10 chaves (`min_length=1, max_length=10`) — índice com dezenas de campos vindo de uma UI é sintoma, não requisito.

### 3. Filtros de query

Filtro que chega pro `explain` rejeita os operadores que **executam JavaScript ou expressão arbitrária do lado do servidor**:

```python
_UNSAFE_FILTER_OPERATORS = {"$where", "$function", "$accumulator", "$expr"}
```

A checagem é recursiva sobre o documento inteiro, não só no nível de cima.

**Namespace vindo de UI e indo direto pro driver é injeção esperando acontecer.** Cobre as três guardas com teste — elas rodam sem cluster nenhum, então não têm desculpa pra ficar de fora do CI.

## Explain e Performance Advisor

O Query Profiler devolve as slow queries e o Torre roda o `explain` do lado, mostrando `totalKeysExamined` e o plano — não só o tempo. Tempo sozinho diz que está lento; o `explain` diz por quê, e é o que justifica o índice sugerido logo ao lado.

## Seeds de carga

`populate_workload.py` e `populate_profiler.py` são scripts standalone que geram atividade real no cluster.

Eles existem por um motivo de ensaio: **Performance Advisor e Query Profiler ficam vazios num cluster saudável e ocioso.** Sem carga semeada antes, essas duas páginas aparecem em branco no meio da demo.
