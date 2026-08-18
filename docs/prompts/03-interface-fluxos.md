# Torre — Atlas Control Plane — interface, fluxos e roteiro

> Terceiro dos três prompts. Quem olha essa tela é DBA, SRE ou quem paga a fatura do Atlas.

---

## O tom

**Densidade de informação em vez de espaço em branco**, e número sempre com a fonte ao lado. Não é landing page.

Duas regras que valem pra tudo:

1. **O frontend nunca vê credencial.** O browser só fala com `/api`.
2. **A tela não calcula recomendação.** O React exibe o resultado e a justificativa — não reimplementa a regra.

## Stack

| Item | Escolha | Motivo |
|---|---|---|
| Build | Vite, portas por env (`WEB_PORT` 5290, `API_PORT` 8765) | a máquina roda várias PoVs; as portas precisam ser configuráveis sem editar código |
| UI kit | LeafyGreen | é um painel de operação Atlas — usar o design system do MongoDB é o mínimo |
| Estado | `useState` no `App.jsx` | nove páginas, cada uma busca o próprio dado. Não há estado global de verdade |
| Navegação | estado `active` + sidenav | sem router. Não vale a dependência |
| HTTP | `axios` com `baseURL: '/api'` e timeout de **60s** | a Admin API é lenta; 60s é medido, não chute |
| Markdown | `react-markdown` + `remark-gfm` | as respostas do Claude vêm em markdown com tabela |

Duas coisas que você vai precisar resolver no `vite.config.js`, e eu já sei porque me custaram tempo:

- **`nodePolyfills`** pra `Buffer`, `global` e `process`. Dependências transitivas do LeafyGreen (`readable-stream`, `through`) usam globais do Node que não existem no browser, e o erro que aparece não aponta pra causa.
- **`manualChunks`** separando `leafygreen`, `markdown` e `vendor`. Sem isso o bundle único passa de 500 kB e dispara warning de build; com a separação o cache do browser também aproveita melhor entre deploys.

Usa `strictPort: false` aqui. Diferente das minhas outras PoVs, o Torre pode subir em porta alternativa sem quebrar nada, porque o alvo do proxy vem de `API_PORT` e não da porta do próprio dev server.

`frontend/src/styles.css` com os tokens dark do MongoDB — `--bg-primary`, `--accent`, `--text-pri/sec/muted` — e tipografia Outfit + JetBrains Mono. Mesma paleta das outras PoVs do portfólio.

## As nove páginas

Uma componente por página em `frontend/src/pages/`. Cada uma responde uma pergunta específica:

| Página | Precisa deixar visível |
|---|---|
| **Overview** | a frota inteira numa tela, sem scroll — é o primeiro slide da conversa |
| **Clusters** | tier, região, versão, storage e auto-scaling reais, direto da Admin API |
| **PerformanceAdvisor** | a recomendação do próprio Atlas, com o botão de criar índice ao lado |
| **Profiler** | query lenta com o `explain` do lado, não só o tempo |
| **Health** | alertas abertos e o score, com os problemas que o compõem |
| **Scale** | o número da heurística **e** a justificativa do Claude, separados |
| **FinOps** | custo por cluster e onde está o desperdício |
| **Compare** | dois clusters lado a lado, mesma métrica na mesma linha |
| **Chat** | resposta em streaming sobre dado real da Admin API, com histórico persistido |

## Contrato com o backend

Todo acesso passa por `src/api.js`, uma função por endpoint. O token opcional entra como header no cliente axios.

| Grupo | Endpoints |
|---|---|
| Frota | `/config`, `/clusters`, `/alerts`, `/invoice` |
| Por cluster | `/cluster/{project_id}/{cluster_name}/…` (PA, slow queries, measurements, series, health) |
| Escala | `GET /scaling`, `POST /scale` |
| Otimização | `/explain`, `/index` |
| FinOps | `/finops` |
| Chat | `/chat`, `/analyze`, `/chat/conversations/…` |
| Relatório | `/report` |

O par `project_id` + `cluster_name` é a chave em quase toda rota. Faz um `_picker.jsx` compartilhado, pra garantir que todas as páginas estão falando do mesmo cluster — nada pior que comparar métrica de um cluster com custo de outro sem perceber.

## Streaming do chat

`axios` cuida dos GETs, mas as duas rotas de streaming (`/chat`, `/analyze`) precisam de `fetch` cru com `getReader()`, porque o `axios` não expõe `ReadableStream` no browser. Acumula o texto e entrega por callback, pra página renderizar markdown parcial enquanto o token chega.

`downloadReport` também usa `fetch` — a resposta é PDF binário, não JSON.

## O roteiro que eu preciso conseguir executar no fim

1. **Overview** — a frota inteira numa tela: quantos projetos, quantos clusters, o que está alertando agora.
2. **Clusters + FinOps** — onde está o dinheiro e quais clusters estão fora de proporção com a carga.
3. **Scale** — mostrar a recomendação e deixar claro, apontando na tela, que o número é heurística determinística e o texto é o modelo. E apontar qual janela sustentou a decisão.
4. **Performance Advisor + Profiler** — o que o Atlas já está apontando, consolidado por cluster em vez de espalhado por telas.
5. **Chat** — perguntar "qual cluster me custa mais e está mais ocioso?" e ver a resposta em streaming sobre dado real. E, se der, perguntar algo de 24h num cluster que só tem janela de 5 minutos, pra mostrar o assistente recusando em vez de estimar.
6. **Relatório PDF** — o artefato que sai da conversa e vai pra reunião de custo.

Um aviso de ensaio: **Performance Advisor e Query Profiler ficam vazios num cluster saudável e ocioso.** Se o roteiro depender deles, ou você semeia carga antes com os scripts de `populate_*`, ou escolhe outra página. Tela vazia no meio da demo é pior que uma página a menos.

## Nota sobre capturas de tela

O repositório é público e os nomes reais de projeto e cluster **identificam um cliente**. Eles aparecem no header, no `<select>` de cluster de todas as páginas, na tabela de FinOps e no pill de contexto do chat. **Substitui no DOM — nós de texto e rótulos de `<option>` — imediatamente antes de cada captura**, não corta depois.

## Antes de apresentar

- `populate_workload.py` e `populate_profiler.py` rodados, senão duas páginas ficam vazias.
- Uma pergunta de aquecimento no chat, pra pagar o cold start fora da demo.
- Nomes de cliente já substituídos, se for capturar tela.
