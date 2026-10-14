# Template de System Design: Aula 02 - APIs Resilientes

> **Instruções:** Faça uma cópia deste arquivo e preencha as seções abaixo com as conclusões da sua pesquisa e os debates. O foco é a integração da API de Checkout com o Serviço de Pagamento Externo para suportar 10.000 usuários simultâneos.

## 1. Escolha de Paradigma da API

- **A Escolha:** ( ) Spring MVC com Virtual Threads ou ( ) Spring WebFlux (Reativo)?
- **Justificativa:** Considerando a capacidade de suportar 10k acessos concorrentes, por que você fez essa escolha técnica?
  > [Sua resposta aqui]

## 2. Desenho de Resiliência

Dependemos de uma API terceira instável. Quais recursos do padrão de resiliência você vai utilizar?

- **Proteção contra Falhas e Lentidão:**
  Quais padrões (Circuit Breaker, TimeLimiter, RateLimiter, Bulkhead, Retry) você implementaria na chamada para o Parceiro de Pagamentos? Por que?
  > [Sua resposta aqui]

- **Proteção contra Duplicidade:**
  Como a `Idempotency-Key` será utilizada na requisição do Checkout para evitar cobranças duplas? Qual o ciclo de vida dessa chave?
  > [Sua resposta aqui]

## 3. Os Edge Cases (Engenharia do Caos)

Cole aqui as 3 perguntas de risco que você formulou e defina como a arquitetura responde a cada uma delas.

1. **E se...** o Parceiro de Pagamentos processar a cobrança, mas a resposta não chegar (timeout ou queda de conexão) e não soubermos se o cliente foi cobrado?
   - **Resposta do Sistema:** O pedido não pode ser marcado nem como falho (o cliente tentaria de novo e seria cobrado duas vezes) nem como pago (ele poderia receber sem ter pago).
     - O pedido vai para o estado `PAGAMENTO_PENDENTE`.
     - A chamada ao parceiro sempre leva a mesma `Idempotency-Key` do pedido. Com isso, o Retry é seguro: se a cobrança já tinha sido feita, o parceiro devolve o mesmo resultado em vez de cobrar de novo.
     - Se as tentativas acabarem, um job de reconciliação consulta o status no parceiro (ou o webhook de confirmação do parceiro avisa) e o pedido é atualizado.
     - O cliente vê "pagamento em processamento", nunca uma cobrança dupla.

2. **E se...** o cliente clicar duas vezes em "Finalizar compra" (ou o app reenviar a requisição) e as duas chegarem ao mesmo tempo, em instâncias diferentes da API?
   - **Resposta do Sistema:** As duas requisições levam a mesma `Idempotency-Key`, gerada uma vez quando o pedido é criado.
     - Antes de processar, cada instância tenta gravar a chave de forma atômica (`SET NX` no Redis ou constraint `UNIQUE` no banco). Só uma consegue e segue para o pagamento.
     - A outra recebe `409 Conflict` enquanto a primeira está em andamento. Se repetir depois que a primeira terminar, recebe a resposta que ficou salva.
     - Se chegar a mesma chave com um corpo diferente, a resposta é `422 Unprocessable Entity`, porque é uso indevido da chave.
     - A chave expira depois de um TTL (por exemplo, 24 horas).

3. **E se...** o Parceiro de Pagamentos ficar fora do ar por 30 minutos no meio de um pico com 10.000 usuários simultâneos?
   - **Resposta do Sistema:** A queda do parceiro não pode derrubar o Checkout inteiro.
     - **Circuit Breaker:** abre depois das primeiras falhas. As chamadas seguintes vão direto para o fallback sem esperar o timeout, evitando milhares de requisições penduradas.
     - **Bulkhead:** limita quantas chamadas ao parceiro rodam ao mesmo tempo, deixando threads e conexões livres para o resto do sistema (carrinho, catálogo, frete).
     - **Fallback:** o pedido é aceito como `PAGAMENTO_PENDENTE` e gravado numa fila (padrão Outbox). O cliente é avisado de que a confirmação chega por e-mail, e a venda não se perde.
     - **Recuperação:** de tempos em tempos o circuito fica meio-aberto e deixa algumas chamadas de teste passarem. Se derem certo, o circuito fecha e um worker processa a fila respeitando o RateLimiter, para o parceiro não cair de novo com uma avalanche de requisições acumuladas.
     - O estado do Circuit Breaker fica na memória de cada instância (Resilience4j), então não depende de banco para funcionar.

## 4. Testando a Resiliência

- **Estratégia:** Como você vai garantir que o Circuit Breaker abre na hora certa e o Timeout não deixa a Thread presa?
  > [Sua resposta aqui. Exemplo: simular latência de 10s no parceiro usando XYZ...]
