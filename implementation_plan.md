# Torre — Atlas Control Plane — prompt de construção

> Esse é o briefing que eu entrego **antes de existir uma linha de código**. Não é documentação do que existe: é o que eu daria pra alguém (ou pro Claude) subir a PoV inteira do zero.

Um dashboard operacional para frotas MongoDB Atlas, com assistente Claude embutido: consolida numa tela só o que hoje está espalhado em seis telas da UI do Atlas — custo, dimensionamento, Performance Advisor, Profiler, alertas — e deixa perguntar em linguagem natural sobre o dado real da Admin API v2. Backend FastAPI em `:8765`, frontend Vite/React/LeafyGreen em `:5290`.

A regra que sustenta tudo: **o número vem da heurística determinística e testada. O modelo escreve a justificativa.**

| Arquivo | O que responde |
|---|---|
| [`docs/prompts/01-arquitetura.md`](docs/prompts/01-arquitetura.md) | os seis módulos, a heurística de escala e a armadilha da janela de tempo, segurança da chave da Admin API, o único ponto de escrita, o assistente, como rodar, ordem de trabalho |
| [`docs/prompts/02-mongodb.md`](docs/prompts/02-mongodb.md) | os três papéis do MongoDB aqui, os índices da memória de chat, as guardas de namespace/chave/filtro, explain e os seeds de carga |
| [`docs/prompts/03-interface-fluxos.md`](docs/prompts/03-interface-fluxos.md) | as nove páginas, contrato de API, streaming do chat, roteiro de demo, nota de capturas |

Se for ler só um: o **01**, pela heurística. Se o LLM entrar antes dela existir, alguém deixa o modelo chutar o tier — e aí a ferramenta perde o argumento inteiro.
