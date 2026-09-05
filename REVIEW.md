# Revisão de engenharia e design — torre-atlas-control-plane

## Resultado

Inicialização dos pools MongoDB da API e memória serializada; regressão concorrente para memória.

Branch `review/codex-improvements`, criada de `main` em `ffa6406ae70730aea9a36c995d58fca72f79b7a3`. Sem merge, push, troca de biblioteca core, alteração de schema ou dataset.

## Commits de correção

- `75f57e3 fix: serialize MongoDB pool creation in API and chat history`

## Commits visible-change

Nenhum.

## Validação

- 33 testes unitários passaram.
- Build de produção passou; análise Ruff E9/F63/F7/F82 com target Python 3.12 passou.
- Browser com APIs bloqueadas: 1440×1000, 768×1024 e 360×800; sem pageerror e sem overflow horizontal no shell inicial; link de salto transfere foco ao conteúdo.
- As 14 cópias de pov-signature.css permanecem idênticas; lang pt-BR confirmado. Nenhuma alteração na camada compartilhada de CSS.
- Auditor de portas passou: registro e configurações alinhados.
- npm audit do lockfile após correções: 0 altos, 0 críticos, 0 moderados e 0 baixos.

## Sugestões não aplicadas e limites

- `api.py:get_client` ainda mantém cache do cliente Atlas/requests.Session separado do cache MongoDB; revisar sincronização/renovação de credenciais em chamadas concorrentes.
- Starlette instalado tem advisories; atualização de framework core ficou como sugestão.
- Não executadas ações de criação de índices/redimensionamento nem chamadas LLM. Nomes de cliente não foram publicados em screenshots.

A verificação visual cobre o shell offline e abas acessíveis sem backend, não todos os estados de dados. Não certifica contraste de cada componente, comportamento touch completo ou toda a navegação com Atlas. Fluxos reais de escrita/carga não foram executados para preservar datasets. Nenhuma comparação de performance foi inventada. Evidências locais: `/tmp/codex-portfolio-review/`.

## Dependências Python

Auditoria do ambiente instalado, não de uma resolução limpa do manifesto; ferramentas de desenvolvimento podem aparecer junto com runtime. Os IDs abaixo não equivalem a exploração confirmada na PoV. Reconciliar versões instaladas/manifests e testar compatibilidade; atualizações core/major ficaram fora desta rodada. Pacotes de ferramenta e componentes extras do venv também não foram alterados fora da branch.

| Pacote instalado | Versão | Advisory | Versões corrigidas informadas |
|---|---|---|---|
| pip | 26.1 | PYSEC-2026-196, PYSEC-2026-3721 | 26.1.2, 26.2 |
| starlette | 0.52.1 | PYSEC-2026-161, PYSEC-2026-249, PYSEC-2026-248, PYSEC-2026-2281, PYSEC-2026-2280 | 1.0.1, 1.1.0, 1.3.0, 1.3.1 |

## Segredos e compartilhamento

Varredura por padrões de chaves privadas, chaves Anthropic/AWS e URI MongoDB autenticada no histórico Git local alcançável: nenhuma credencial real confirmada; matches encontrados eram placeholders conhecidos. Limite: não é scanner de entropia, não cobre objetos inacessíveis, texto em screenshots nem logs externos.

Nenhum import/referência estática a `_shared/grove_client.py` foi encontrado nesta PoV. Configuração própria de gateway/ambiente não constitui dependência de código desse módulo. `_shared` permaneceu intocado; consumidores externos/dinâmicos não são garantidos por busca estática. Relatório separado: `../REVIEW_SHARED.md`.

## Segunda rodada — melhorias adicionais

Criação do cliente Atlas e sua requests.Session protegida por lock, inclusive renovação por credenciais; cache MongoDB já corrigido anteriormente. 34 testes passaram, incluindo cold start concorrente do cliente Atlas. Nenhuma chamada à Admin API/LLM foi realizada.
