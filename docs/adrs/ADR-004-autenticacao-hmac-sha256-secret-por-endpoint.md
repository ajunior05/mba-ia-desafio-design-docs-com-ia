# ADR-004: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint e rotação

## Status
Aceito

## Contexto
Os webhooks expõem dados de pedidos para endpoints fora da infraestrutura da empresa. O cliente que recebe precisa de uma forma de validar que a requisição realmente partiu da plataforma e que o payload não foi adulterado em trânsito.

Já houve um incidente em que um cliente vazou uma secret em log de aplicação do lado dele — o que reforça a necessidade de limitar o dano de um vazamento a um único endpoint, e de permitir rotação sem downtime da integração.

## Decisão
Assinar o corpo de cada requisição de webhook com **HMAC-SHA256**, enviando a assinatura no header `X-Signature`, para o cliente validar do lado dele.

Cada **endpoint de webhook cadastrado tem sua própria secret**, gerada pela plataforma e devolvida na criação — não existe secret global compartilhada entre todos os endpoints de um customer ou da plataforma.

A secret é **rotacionável** via API. Durante a rotação, a secret antiga permanece válida em paralelo por **24 horas (grace period)**, para o cliente ter tempo de migrar seus sistemas; após esse período, a secret antiga é invalidada.

Adicionalmente (requisitos não funcionais correlatos, tratados como validação e não como decisão arquitetural separada): URL do webhook deve ser obrigatoriamente HTTPS (validação de schema), e o payload tem limite de 64KB — se excedido, o envio falha com erro em vez de truncar.

## Alternativas Consideradas
- **Secret global por plataforma** (compartilhada entre todos os endpoints). Descartada: um único vazamento comprometeria a validação de todos os endpoints de todos os clientes simultaneamente, ampliando o raio de impacto de qualquer incidente.

## Consequências

**Positivas**
- HMAC-SHA256 é padrão de mercado amplamente suportado por bibliotecas client-side, reduzindo fricção de integração.
- Secret por endpoint limita o impacto de um vazamento a uma única integração, em vez de comprometer toda a base de clientes.
- Grace period de 24h permite rotação sem indisponibilidade da integração do lado do cliente.

**Negativas**
- Aumenta a complexidade de modelagem: é necessário armazenar e gerenciar o estado de duas secrets (atual e anterior) por período limitado, com expiração controlada.
- Requer que os clientes implementem corretamente a verificação HMAC do lado deles — erro de implementação no cliente pode gerar falsos negativos de validação.

## Fonte
TRANSCRICAO.md — [09:19] a [09:24] (Sofia, Bruno, Diego, Larissa).
