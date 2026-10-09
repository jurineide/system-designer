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
   - **Resposta do Sistema:** O pedido **não** é marcado como falho nem como pago. Ele vai para o estado `PAGAMENTO_PENDENTE`. A chamada ao parceiro sempre leva a mesma `Idempotency-Key` do pedido, então um Retry é seguro: se a cobrança já foi feita, o parceiro devolve o mesmo resultado em vez de cobrar de novo. Se os retries se esgotarem, um job de reconciliação consulta o status no parceiro (ou recebe o webhook de confirmação) e atualiza o pedido. O cliente vê "pagamento em processamento", nunca uma cobrança dupla.

2. **E se...** o cliente clicar duas vezes em "Finalizar compra" (ou o app reenviar a requisição) e as duas chegarem ao mesmo tempo, em instâncias diferentes da API?
   - **Resposta do Sistema:** As duas requisições levam a mesma `Idempotency-Key`. Antes de processar, cada instância tenta gravar a chave de forma atômica (`SET NX` no Redis ou constraint `UNIQUE` no banco). Só uma consegue e segue para o pagamento. A outra recebe `409 Conflict` enquanto a primeira está em andamento e, depois que ela termina, recebe a resposta salva. Se chegar a mesma chave com um corpo diferente, a resposta é `422 Unprocessable Entity`.

3. **E se...** o Parceiro de Pagamentos ficar fora do ar por 30 minutos no meio de um pico com 10.000 usuários simultâneos?
   - **Resposta do Sistema:** O Circuit Breaker abre depois das primeiras falhas e as chamadas seguintes vão direto para o fallback, sem esperar timeout. O Bulkhead limita as chamadas simultâneas ao parceiro, então as threads e conexões do resto do Checkout (carrinho, catálogo, frete) continuam livres. No fallback, o pedido é aceito como `PAGAMENTO_PENDENTE` e gravado numa fila (padrão Outbox), e o cliente é avisado de que a confirmação chega por e-mail. Quando o circuito passa para meio-aberto e o parceiro volta, um worker processa a fila respeitando o RateLimiter, para não derrubar o parceiro com uma avalanche de requisições. O estado do Circuit Breaker fica em memória em cada instância (Resilience4j), então ele não depende de banco para funcionar.

## 4. Testando a Resiliência

- **Estratégia:** Como você vai garantir que o Circuit Breaker abre na hora certa e o Timeout não deixa a Thread presa?
  > [Sua resposta aqui. Exemplo: simular latência de 10s no parceiro usando XYZ...]
