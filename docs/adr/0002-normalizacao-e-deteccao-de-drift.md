# ADR 0002: Normalização De Dados Sociais E Detecção De Drift

## Status

Aceito.

## Contexto

A extensão coleta dados de redes sociais com formatos diferentes e instáveis. No Instagram, parte dos dados vem de payloads internos, parte de SSR/hydration e parte do DOM visível. No X, os dados vêm principalmente de payloads GraphQL.

Para ingestão no Heart Hub, não devemos enviar o JSON cru de cada provider diretamente para a API como dado de produto. O raw payload deve ser preservado para auditoria e reprocessamento, mas a API deve receber dados processados, deduplicados e normalizados.

Também precisamos detectar quando um provider muda o formato esperado da DOM, dos payloads ou da forma de coleta. Essa detecção deve ser genérica o suficiente para ser reutilizada por Instagram, X e futuros providers.

## Decisão

Adotaremos uma pipeline em duas camadas:

1. **Camada de adaptação por provider**
   - Responsável por entender estruturas específicas de cada plataforma.
   - Extrai dados brutos para entidades sociais canônicas, como `SocialPublication`, `SocialComment` e `SocialEngagement`.
   - Pode usar payloads, SSR, DOM ou outros sinais específicos do provider.
   - Deve preservar o raw payload quando existir.

2. **Camada de pipeline agnóstica**
   - Responsável por processamentos comuns a todos os providers.
   - Deduplicação por chaves canônicas.
   - Garantia de metadados comuns, como `captured_at`.
   - Validação de contrato mínimo.
   - Registro de qualidade/confiança.
   - Detecção e exportação de drift de formato.
   - Preparação do JSON final para ingestão no Heart Hub.

Na prática, os providers não devem “decidir o produto final” sozinhos. Eles adaptam formatos externos para o domínio interno. A pipeline agnóstica decide como consolidar, validar e expor os dados.

## Alternativas Avaliadas

### Normalização totalmente dentro de cada provider

Vantagens:

- Mais simples no curto prazo.
- Cada parser resolve tudo no mesmo lugar.
- Menos abstração inicial.

Desvantagens:

- Duplica deduplicação, validação e tratamento de qualidade.
- Aumenta divergência entre providers.
- Torna mais difícil garantir um contrato único para o Heart Hub.
- Faz cada integração crescer como um sistema isolado.

Conclusão: útil para extração específica, mas insuficiente como arquitetura principal.

### Normalização totalmente agnóstica, sem lógica por provider

Vantagens:

- Contrato centralizado.
- Menos acoplamento com plataformas.

Desvantagens:

- Irrealista para redes sociais com estruturas muito diferentes.
- A pipeline genérica não consegue inferir sozinha campos específicos como shortcode, thread, quote, reel, carousel, likers ou reply context.
- Levaria a heurísticas frágeis e difíceis de depurar.

Conclusão: desejável como destino de consolidação, mas não substitui adapters por provider.

### Pipeline híbrida com adapters por provider e processamento agnóstico

Vantagens:

- Mantém conhecimento específico isolado.
- Permite contrato comum para o Heart Hub.
- Facilita novos providers.
- Centraliza deduplicação, timestamps, qualidade, drift e export.
- Preserva raw payload para auditoria e reprocessamento.

Desvantagens:

- Exige disciplina para não misturar regra de produto dentro do adapter.
- Exige tipos canônicos bem mantidos.
- Pode demandar versionamento de schema conforme a integração amadurece.

Conclusão: escolhida.

## Drift De Formato

A extensão passa a exportar `format_drift_issues`.

Um drift representa uma suspeita de que a estrutura esperada mudou ou de que a coleta perdeu qualidade. Exemplos:

- DOM visível existe, mas nenhum link canônico de publicação foi extraído.
- Publicações visíveis foram extraídas, mas a maioria não possui autor.
- Um payload conhecido de feed/publicação foi capturado, mas nenhum item foi normalizado.
- Um endpoint suportado deixou de produzir entidades.

Cada issue deve conter:

- `provider`
- `detector`
- `severity`
- `expected`
- `observed`
- `page_url`
- `captured_at`
- `details`

Esse formato é genérico e pode ser emitido por qualquer provider ou etapa da pipeline.

## Metadados De Captura

Entidades normalizadas devem carregar `captured_at`.

Esse campo representa quando a extensão capturou ou normalizou aquele dado, não necessariamente quando o conteúdo foi publicado na rede social. Para horário original do conteúdo, usamos campos como `created_at` ou `engaged_at`.

## Consequências

- O Heart Hub deve consumir prioritariamente entidades normalizadas.
- Raw payloads continuam disponíveis para auditoria, debugging e reprocessamento.
- O export fica mais confiável para ingestão e para troubleshooting.
- A equipe consegue identificar mudanças silenciosas no Instagram/X antes que a ingestão gere dados incorretos.
- Futuras integrações devem implementar adapters para o domínio canônico, não exports próprios incompatíveis.

