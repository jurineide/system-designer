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
   - **Resposta do Sistema:** [Como o sistema deve se comportar?]

2. **E se...** o cliente clicar duas vezes em "Finalizar compra" (ou o app reenviar a requisição) e as duas chegarem ao mesmo tempo, em instâncias diferentes da API?
   - **Resposta do Sistema:** [Como o sistema deve se comportar?]

3. **E se...** o Parceiro de Pagamentos ficar fora do ar por 30 minutos no meio de um pico com 10.000 usuários simultâneos?
   - **Resposta do Sistema:** [Como o sistema deve se comportar?]

## 4. Testando a Resiliência

- **Estratégia:** Como você vai garantir que o Circuit Breaker abre na hora certa e o Timeout não deixa a Thread presa?
  > [Sua resposta aqui. Exemplo: simular latência de 10s no parceiro usando XYZ...]
