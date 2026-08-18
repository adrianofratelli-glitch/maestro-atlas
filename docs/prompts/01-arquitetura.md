# Torre — Atlas Control Plane — arquitetura e princípios

> Primeiro dos três prompts que eu uso pra levantar essa PoV do zero. O problema, a heurística que sustenta a credibilidade, a segurança e o assistente. Dado, índices e guardas em `02-mongodb.md`; tela e roteiro em `03-interface-fluxos.md`.

---

## O que eu quero construir

O **Torre**: um dashboard operacional para frotas MongoDB Atlas, com um assistente Claude embutido.

O problema que ele resolve é bem concreto. Quem opera várias organizações e projetos Atlas passa o dia navegando entre telas da própria UI do Atlas pra responder pergunta simples: *quais clusters estão superdimensionados? o que o Performance Advisor está apontando? quanto isso está custando e onde está o desperdício?* A resposta existe, mas está espalhada em seis telas diferentes e três níveis de navegação.

O Torre consolida isso numa tela só e deixa perguntar em linguagem natural, com o modelo enxergando os dados reais da Admin API — não um resumo pré-mastigado.

## Arquitetura

```
React 18 + Vite + LeafyGreen (:5290) --proxy /api--> FastAPI (:8765)
                                                       |-> atlas_client.py  -> Atlas Admin API v2
                                                       |-> ai_agent.py      -> Anthropic Claude
                                                       \-> chat_memory.py   -> MongoDB Atlas
```

Módulos Python planos na raiz, sem pacote aninhado. São seis arquivos, cada um com uma responsabilidade que dá pra explicar numa frase:

| Arquivo | Papel |
|---|---|
| `api.py` | backend FastAPI, todas as rotas `/api/...`, middleware de request-id e métricas, auth opcional |
| `atlas_client.py` | cliente da Atlas Admin API v2 + a heurística de recomendação de escala |
| `ai_agent.py` | análise Claude, chat com streaming, geração de PDF |
| `chat_memory.py` | histórico de chat persistido no Atlas via pymongo |
| `observability.py` | log estruturado (`LOG_JSON=1`) e métricas em processo (`GET /api/metrics`) |
| `populate_workload.py`, `populate_profiler.py` | seeds standalone de dado de exemplo |

## A regra que sustenta a PoV inteira

**O número vem da heurística determinística. O modelo escreve a justificativa.**

A recomendação de escala — subir, descer ou manter o tier — sai de uma função em `atlas_client.py`, testada com unittest. Não é o Claude olhando métrica e chutando tier. O Claude recebe a recomendação pronta e traduz ela em argumento de negócio, pra quem paga a conta entender.

Isso importa porque a pergunta que **sempre** vem é: *"esse tier aí foi o modelo que chutou?"*. A resposta precisa ser "não, e tem teste unitário cobrindo a regra" — e a tela precisa deixar isso óbvio antes mesmo de alguém perguntar. Na página Scale, separa o número da justificativa **literalmente na diagramação**. Não mistura os dois num parágrafo.

### A heurística, e a armadilha da janela de tempo

A regra olha cinco sinais, cada um com dois patamares (atenção e crítico), e o mais severo vence:

- **CPU** — 65% avisa, 80% é gargalo.
- **Memória** — 75% avisa (risco de page fault), 90% é working set que não cabe na RAM, ou seja, pressão de cache do WiredTiger.
- **Storage** — 70% planeja expansão, 85% é risco de esgotar disco.
- **Conexões** — como percentual do limite **do tier**, não em número absoluto. 60% avisa, 80% é crítico.
- **IOPS** — acima de 3.000 num tier não-NVMe, sugere avaliar NVMe.

E aí vem o detalhe que eu considero o mais importante da função inteira:

> **Decisão de escala não pode sair de um instantâneo de 5 minutos.** Uma demo às 9 da manhã recomendaria diminuir um cluster que tem pico às 3 da madrugada.

Então: quando existe a série de 24h, o **scale up olha o p95** e o **scale down olha a média**. Assimétrico de propósito — pra subir, o que importa é o pico; pra descer, o que importa é o comportamento típico. E a UI precisa dizer qual janela foi usada ("CPU (p95 24h)" contra só "CPU"), porque a mesma recomendação com base diferente é uma recomendação diferente.

O scale down também exige **todos os sinais baixos ao mesmo tempo** (CPU média < 15%, conexões < 20%, memória < 50%, disco < 50%) e nunca sugere descer de M10. Um único critério de ociosidade recomendaria encolher um cluster que só é ocioso em CPU.

A função devolve `action`, `severity`, a lista de `reasons` já em português com o número embutido, e um bloco `metrics` com os valores crus pra tela mostrar ao lado. Isso não é conveniência: é o que permite alguém conferir a conta na hora.

## Segurança — o ponto inegociável

**A chave da Atlas Admin API dá poder administrativo sobre a organização inteira.** Ela vive só no `.env` do backend, e o frontend nunca a vê. Em nenhuma circunstância ela passa por um bundle de browser.

Além disso:

- Autenticação por bearer token **opcional**, via `API_AUTH_TOKEN`, com comparação em tempo constante. Deixa desligada por padrão pra uso local, mas documenta que qualquer uso além disso liga. Com ela ligada, **todo** endpoint `/api` não-health e o `/metrics` passam a exigir o token.
- Guardas de injeção e validação de id no `chat_memory`, e validação de namespace/chaves na criação de índice — detalhadas em `02-mongodb.md`. **Cobre as duas com teste.**
- O health expõe só estado, nunca detalhe de conexão.
- O escopo sobre a Admin API é de **leitura**, exceto por um único ponto.

### O único ponto de escrita

`scaleCluster` é a única função do frontend que muda estado no Atlas de verdade. Isola ela numa função só, com esse nome, justamente por isso: qualquer revisão de segurança tem um lugar único pra olhar.

Todo o resto é análise e recomendação.

## O cliente da Admin API

Três coisas que não são detalhe:

- **Cache com TTL nas chamadas caras.** Descobrir qual processo é o primário de um cluster custa uma sequência de chamadas, e nove páginas pedindo isso a cada render transforma a Admin API no gargalo da própria ferramenta.
- **Métrica vem do primário**, resolvido explicitamente — não do primeiro processo que a lista devolver.
- **Ponto nulo é ponto nulo.** A Admin API devolve buracos nas séries; pega o último valor não-nulo em vez de assumir zero. Zero e "não reportou" são coisas diferentes, e tratar as duas igual produz uma recomendação de scale down linda e errada.

E a regra geral: **não inventa métrica.** Se a Admin API não devolve, a tela mostra que não devolve. Isso vale inclusive pro assistente — quando alguém pergunta sobre 24h e só existe janela de 5 minutos, a resposta certa é dizer que só tem 5 minutos. Isso não é falha pra esconder; é exatamente o comportamento que faz a ferramenta ser confiável.

## Assistente Claude

`ai_agent.py` faz três coisas:

- **Análise** — interpreta o estado de custo, performance e status vindo da Admin API.
- **Chat com streaming** — resposta token a token.
- **Geração de relatório PDF** — o artefato que sai da conversa e vai pra reunião de custo. Esse é o entregável que faz a ferramenta ser usada de novo na semana seguinte.

Modelo default: Sonnet 5, configurável por `CLAUDE_MODEL`. Histórico persistido no Atlas via `chat_memory.py`.

## Comandos que eu preciso ter

```bash
./run_react.sh   # ativa a venv, instala dependências se preciso,
                 # valida as portas reservadas, sobe backend + frontend
```

Isolados:

```bash
uvicorn api:app --reload --port 8765
cd frontend && npm run dev      # :5290, proxia /api -> :8765
cd frontend && npm run build
python -m unittest discover -s tests -v   # sem credencial Atlas/Mongo
python populate_workload.py
python populate_profiler.py
```

Docker em container único: nginx serve o build (com CSP e headers de segurança) e proxia `/api` pro FastAPI, rodando non-root.

```bash
docker build -t torre .
docker run --env-file .env -p 18085:8080 torre
```

Os testes **não podem exigir credencial** — só lógica pura e guardas. Se um teste começar a precisar da Admin API, ele deixou de ser teste e virou verificação manual. Não tem linter configurado neste repo, e tudo bem.

## Ambiente

| Variável | Obrigatória | Papel |
|---|---|---|
| `ATLAS_PUBLIC_KEY` | sim | chave pública da Admin API |
| `ATLAS_PRIVATE_KEY` | sim | chave privada |
| `ATLAS_ORG_ID` | sim | organização a inspecionar |
| `ANTHROPIC_API_KEY` | sim | assistente e relatórios |
| `MONGODB_URI` | não | criação de índices + histórico de chat |
| `CLAUDE_MODEL` | não | default Sonnet 5 |
| `API_AUTH_TOKEN` | não | liga autenticação por bearer token |

## Como quero que você trabalhe

- UI em **pt-BR** (público brasileiro). Código-fonte, comentários e documentação em **inglês**.
- Nenhuma lógica de decisão no React. Se a tela está calculando alguma coisa que o backend deveria ter calculado, está errado.
- Toda regra determinística tem teste. A heurística de escala é a mais importante — é ela que sustenta a credibilidade da ferramenta inteira.
- Não inventa métrica. Se a Admin API não devolve, a tela mostra que não devolve — e o assistente diz isso em voz alta em vez de estimar.
- Onde a decisão depende da janela de tempo, a janela aparece na tela junto do número.

## Ordem de trabalho

1. `atlas_client.py` — cliente da Admin API, com as chamadas de leitura funcionando via `curl`-equivalente antes de existir qualquer tela.
2. A heurística de escala, **com os testes**, antes de qualquer LLM entrar na história. Inclusive o caminho de 24h contra o de 5 minutos.
3. `api.py` expondo tudo isso.
4. Observabilidade: request-id, métricas, log estruturado.
5. Páginas de leitura pura: Overview, Clusters, Health, FinOps, Compare.
6. Scale — a página que junta heurística e justificativa.
7. `ai_agent.py` e o chat com streaming.
8. `chat_memory.py` com as guardas e os testes delas.
9. Relatório PDF.

O passo 2 vem antes do 7 de propósito. Se o LLM entrar antes da heurística existir, a tentação de deixar ele estimar o tier é grande demais — e aí a PoV perde exatamente o argumento que ela existe pra fazer.

## Fronteiras — não gasta tempo com isso

- Somente leitura sobre a Admin API, com a exceção única e isolada do `scaleCluster`.
- Autenticação desligada por padrão, feita pra uso local do time de operação.
- Métricas de observabilidade são em processo e resetam no restart.
- Uma única `ATLAS_ORG_ID`. Multi-org com RBAC é conversa de produção, não de PoV.
