# Torre — Atlas Control Plane — interface, fluxos e roteiro

> Terceira parte do briefing. Quem olha essa tela é DBA, SRE ou quem paga a fatura do Atlas.

---
## Estado atual — modo palco

A navegação primária tem cinco destinos: **Overview**, **Saúde**, **Escala**,
**FinOps** e **Assistente**. Clusters, Performance Advisor, Profiler e comparação
deixaram o menu principal; a operação continua disponível nos módulos que
consomem esses dados. A regra agora é densidade relevante, não densidade total.

## Contrato visual do portfólio (v2)

Esta UI participa da assinatura MongoDB Dark das PoVs. O arquivo
`src/pov-signature.css` é uma cópia sincronizada entre os onze frontends e deve
ser importado **depois** do stylesheet local. O contêiner raiz carrega
`data-pov-shell`, existe um `.pov-skip-link` para `#conteudo-principal` e o
`index.html` declara pt-BR, dark color scheme, theme color e o favicon comum.

A camada compartilhada é dona da document rail, foco, touch targets e redução de
movimento. Este arquivo continua dono do fluxo e das exceções de domínio: não
achate uma tela operacional num template de landing page e não remova a tese
visual específica desta PoV. Qualquer mudança na assinatura precisa ser
replicada nas onze cópias e validada em 1440, 768 e 360 px, além do build de
produção e do estado offline.


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
| Estado | `useState` no `App.jsx` | cinco páginas primárias, cada uma busca o próprio dado. Não há estado global de verdade |
| Navegação | estado `active` + menu compacto | sem router. Não vale a dependência |
| HTTP | `axios` com `baseURL: '/api'` e timeout de **60s** | a Admin API é lenta; 60s é medido, não chute |
| Markdown | `react-markdown` + `remark-gfm` | as respostas do Claude vêm em markdown com tabela |

Duas coisas que você vai precisar resolver no `vite.config.js`, e eu já sei porque me custaram tempo:

- **`nodePolyfills`** pra `Buffer`, `global` e `process`. Dependências transitivas do LeafyGreen (`readable-stream`, `through`) usam globais do Node que não existem no browser, e o erro que aparece não aponta pra causa.
- **`manualChunks`** separando `leafygreen`, `markdown` e `vendor`. Sem isso o bundle único passa de 500 kB e dispara warning de build; com a separação o cache do browser também aproveita melhor entre deploys.

Usa `strictPort: false` aqui. Diferente das minhas outras PoVs, o Torre pode subir em porta alternativa sem quebrar nada, porque o alvo do proxy vem de `API_PORT` e não da porta do próprio dev server.

`frontend/src/styles.css` com os tokens dark do MongoDB — `--bg-primary`, `--accent`, `--text-pri/sec/muted` — e tipografia Outfit + JetBrains Mono. Mesma paleta das outras PoVs do portfólio.

## As cinco páginas primárias

Uma componente por página em `frontend/src/pages/`. Cada uma responde uma pergunta específica:

| Página | Precisa deixar visível |
|---|---|
| **Overview** | a frota inteira numa tela, sem scroll — é o primeiro slide da conversa |
| **Health** | alertas abertos e o score, com os problemas que o compõem |
| **Scale** | o número da heurística **e** a justificativa do Claude, separados |
| **FinOps** | custo por cluster e onde está o desperdício |
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
2. **FinOps** — onde está o dinheiro e quais clusters estão fora de proporção com a carga.
3. **Scale** — mostrar a recomendação e deixar claro, apontando na tela, que o número é heurística determinística e o texto é o modelo. E apontar qual janela sustentou a decisão.
4. **Saúde** — alertas e sinais operacionais consolidados, sem abrir páginas de diagnóstico separadas.
5. **Chat** — perguntar "qual cluster me custa mais e está mais ocioso?" e ver a resposta em streaming sobre dado real. E, se der, perguntar algo de 24h num cluster que só tem janela de 5 minutos, pra mostrar o assistente recusando em vez de estimar.
6. **Relatório PDF** — o artefato que sai da conversa e vai pra reunião de custo.

Performance Advisor e Query Profiler permanecem como fontes auxiliares; não são
destinos do roteiro principal. Se forem usados numa pergunta, semeie carga com
os scripts `populate_*` antes da apresentação.

## Nota sobre capturas de tela

O repositório é público e os nomes reais de projeto e cluster **identificam um cliente**. Eles aparecem no header, no `<select>` de cluster de todas as páginas, na tabela de FinOps e no pill de contexto do chat. **Substitui no DOM — nós de texto e rótulos de `<option>` — imediatamente antes de cada captura**, não corta depois.

## Antes de apresentar

- `populate_workload.py` e `populate_profiler.py` rodados, senão duas páginas ficam vazias.
- Uma pergunta de aquecimento no chat, pra pagar o cold start fora da demo.
- Nomes de cliente já substituídos, se for capturar tela.
