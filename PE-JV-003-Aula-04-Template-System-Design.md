# Template de System Design: Aula 04 - Microsserviços & Padrão Saga

> **Contexto do Cenário: O Pedido Pago Sem Produto na Black Friday**
>
> Sua equipe está arquitetando a transação distribuída de compra na Black Friday para o marketplace:
>
> 1. **O Incidente:** Um cliente comprou uma Smart TV de R$ 3.000 via Pix. O gateway de pagamentos debitou o valor imediatamente e confirmou a transação. No milissegundo seguinte, o serviço de Estoque esgotou as unidades e rejeitou a reserva. O dinheiro sumiu da conta do cliente, o produto não existe para entrega e o pedido travou em um estado inconsistente.
> 2. **O Desafio Distribuído:** Não existe transação única (`@Transactional`) entre bancos de dados autônomos de Checkout, Pagamento e Estoque.
> 3. **A Carga:** 5.000 TPS de checkout simultâneo e milhões de clientes consultando o status da compra em "Meus Pedidos".
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Transações Distribuídas: Coordenação e Trade-offs

### 1.1 Volumetria e Pragmatismo: 2PC vs Saga

- **Análise Crítica:** Em quais cenários arquiteturais uma transação ACID local ou Two-Phase Commit (2PC) resolveria o problema com menor custo operacional? Por que, para o cenário de 5.000 TPS na Black Friday com serviços e bancos autônomos, o 2PC é inviável e a consistência eventual se torna necessária?
  > - **ACID local resolve** quando os dados estão no mesmo banco (ex.: monolito modular com Checkout, Pagamento e Estoque num só PostgreSQL): um único `@Transactional` e pronto.
  > - **2PC resolve** com poucos participantes, baixo volume, todos no mesmo datacenter e com suporte a XA (ex.: integração interna de sistemas legados).
  > - **Por que 2PC é inviável aqui:**
  >   - é bloqueante: todos os participantes seguram locks até o coordenador decidir, então a latência vira a do participante mais lento e, a 5.000 TPS, isso esgota conexões;
  >   - o coordenador é ponto único de falha: se cair depois do *prepare*, os recursos ficam travados;
  >   - a disponibilidade total vira o produto da disponibilidade de cada serviço;
  >   - o gateway de Pix e o broker não participam de XA.
  >
  > Por isso usamos **Saga**: uma sequência de transações locais, cada uma com sua compensação, aceitando consistência eventual.

### 1.2 O Modelo de Coordenação da Saga

- **A Escolha:** ( ) Coreografia (Orientada a Eventos)  (X) Orquestração (Process Manager / Máquina de Estados)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Coreografia:* Como seu design resolve a falta de visibilidade centralizada (saber em tempo real onde o pedido está) e previne dependências cíclicas à medida que novos serviços entrarem no fluxo de compra?
  - *Se você escolheu Orquestração:* Qual componente atua como Orquestrador, como ele se comunica com os participantes (síncrono ou assíncrono) e como você evita que ele se transforme em um "serviço Deus", tirando a inteligência e a autonomia dos outros microsserviços?
  > - **Orquestrador:** a `SagaDePedido`, dentro do serviço de Checkout (dono do pedido). É uma máquina de estados persistida no banco do Checkout, o que dá visibilidade em tempo real de onde cada pedido está.
  > - **Comunicação:** assíncrona, por comandos e respostas num broker (Kafka/RabbitMQ): `ReservarEstoque` → `EstoqueReservado`/`EstoqueRejeitado`, `CobrarPagamento` → `PagamentoAprovado`/`PagamentoRecusado`. Um participante lento não prende threads do orquestrador.
  > - **Evitando o "serviço Deus":**
  >   - o orquestrador só conhece a **sequência** e as **transições**; as regras de negócio ficam nos participantes (o Estoque decide se há saldo, o Pagamento fala com o gateway);
  >   - ele nunca acessa o banco de outro serviço, só envia comandos;
  >   - uma saga por processo de negócio, e não um orquestrador central para tudo.

---

## 2. Máquina de Estados, Compensação e Confiabilidade

### 2.1 Máquina de Estados e Fluxos Ponta a Ponta

- **Estados do Pedido:** Defina a máquina de estados explícita do ciclo de vida da transação no seu modelo (incluindo os estados iniciais, intermediários, de compensação e finais).
  > - **Inicial:** `CRIADO`
  > - **Intermediários:** `RESERVANDO_ESTOQUE` → `AGUARDANDO_PAGAMENTO` → `PAGO` → `CONFIRMANDO_ESTOQUE`
  > - **Compensação:** `LIBERANDO_ESTOQUE` (pagamento recusado), `ESTORNANDO` (falha de estoque após o pagamento)
  > - **Finais:** `CONCLUIDO`, `CANCELADO` (sem cobrança), `CANCELADO_COM_ESTORNO`, `FALHA_ESTORNO` (exige tratamento manual)
  >
  > Decisão de design: **reservar o estoque antes de cobrar**. Isso evita a maior parte do incidente, porque o cliente só paga por um item que está separado para ele. A reserva tem prazo de expiração.
- **Detalhamento dos Fluxos:**
  - **Caminho Feliz:** Descreva o fluxo de ponta a ponta da transação concluída com sucesso entre Checkout, Pagamento e Estoque.
  - **Caminho de Falha & Compensação:** Descreva o passo a passo a partir da falha na reserva de estoque após o pagamento aprovado. Como a ação compensatória (estorno) é disparada e executada sem intervenção humana?
  > **Caminho Feliz:**
  > 1. Checkout cria o pedido (`CRIADO`) e a saga envia `ReservarEstoque`.
  > 2. Estoque reserva o item com prazo e responde `EstoqueReservado` → `AGUARDANDO_PAGAMENTO`.
  > 3. A saga envia `CobrarPagamento` (com Idempotency-Key = ID do pedido); o Pagamento confirma o Pix e responde `PagamentoAprovado` → `PAGO`.
  > 4. A saga envia `ConfirmarReserva`; o Estoque baixa a reserva definitivamente → `CONCLUIDO`.
  >
  > **Caminho de Falha & Compensação** (o pagamento foi aprovado, mas o Estoque responde `EstoqueRejeitado` na confirmação, por exemplo porque a reserva expirou durante um Pix lento):
  > 1. A saga recebe `EstoqueRejeitado` e transiciona o pedido para `ESTORNANDO`, gravando o estado no banco.
  > 2. Na mesma transação, grava o comando `EstornarPagamento` na tabela de outbox.
  > 3. O Pagamento consome o comando e pede a devolução do Pix ao gateway, usando a Idempotency-Key `estorno-{pedidoId}`.
  > 4. O Pagamento responde `PagamentoEstornado` → `CANCELADO_COM_ESTORNO`, e o cliente é notificado.
  >
  > Tudo isso é disparado pela própria máquina de estados ao receber o evento de falha, sem intervenção humana.

### 2.2 Resiliência da Compensação e Consistência

- **E se a compensação falhar ou oscilar?** A chamada ou evento de estorno pode sofrer timeout ou instabilidade transitória. Como seu design garante que o cliente não fique no prejuízo se a tentativa de compensação falhar ou for reprocessada?
  > - **Compensação sempre retentável:** o estorno não pode ser recusado por regra de negócio. Se falhar por algo transitório, é reenviado com backoff exponencial.
  > - **Idempotência:** o estorno usa sempre a mesma chave (`estorno-{pedidoId}`), então reprocessar ou receber a mensagem duas vezes nunca devolve o dinheiro duas vezes.
  > - **Estado persistido:** como `ESTORNANDO` está gravado no banco, se o serviço cair no meio, a saga retoma de onde parou ao reiniciar.
  > - **Último recurso:** depois de N tentativas, a mensagem vai para uma DLQ, o pedido vai para `FALHA_ESTORNO` e um alerta é disparado. Um job de reconciliação compara as transações do gateway com os pedidos cancelados e reprocessa o que ficou pendente.
- **Atomicidade entre Estado e Mensageria (Dual Write):** Se a sua arquitetura envolve publicação de eventos ou mensagens entre passos, como você garante que a persistência no banco local e o despacho da mensagem ocorram sem risco de perda silenciosa se a rede ou o broker falharem entre as duas operações?
  > **Transactional Outbox.** Em vez de salvar no banco e depois publicar no broker (duas operações que podem falhar separadamente), o serviço grava, **na mesma transação local**, a mudança de estado e a mensagem numa tabela `outbox`. Um relay (polling ou CDC com Debezium) lê a outbox e publica no broker, marcando como enviada. Se o broker estiver fora, a mensagem continua na tabela e é publicada quando ele voltar. A entrega é *at-least-once*, então cada consumidor é idempotente: guarda os IDs já processados numa tabela `inbox` e ignora duplicatas.

---

## 3. Desacoplamento com CQRS e Estratégia de Testes

### 3.1 Consultas em Alta Vazão com CQRS

- **O Problema de Leitura:** Durante o pico de vendas, milhões de usuários atualizam a página de "Meus Pedidos" a cada poucos segundos.
- **Desenho da Solução:** Como você separa a responsabilidade de escrita (Command) da consulta de status (Query)? Como o modelo de leitura é alimentado e como sua arquitetura lida com a consistência eventual (o delay entre o pagamento e a visualização do status)?
  > - **Command:** criar pedido, pagar e cancelar passam pela saga e gravam nos bancos de escrita (PostgreSQL) de cada serviço.
  > - **Query:** um serviço de consulta separado lê só de um modelo de leitura desnormalizado (ex.: MongoDB ou Redis, indexado por `customer_id`), com um documento por pedido já pronto para a tela. As consultas nunca tocam o banco transacional.
  > - **Alimentação:** os eventos da saga (`PedidoCriado`, `PagamentoAprovado`, `PedidoCancelado`...) saem pela outbox para o Kafka, e um consumidor de projeção atualiza o modelo de leitura. Cada evento leva a versão do pedido, então eventos fora de ordem ou duplicados são ignorados.
  > - **Consistência eventual:**
  >   - a tela mostra estados intermediários honestos ("pagamento em processamento");
  >   - logo após a compra, o front usa o status devolvido pelo próprio comando, sem esperar a projeção;
  >   - em vez de o usuário atualizar a página a cada segundo, o status chega por SSE/WebSocket quando muda, o que também reduz a carga;
  >   - o lag do consumidor é monitorado, com alerta se passar de alguns segundos.

### 3.2 Estratégia de Testes da Transação Distribuída

- **Validação de Resiliência:** Como comprovar em ambiente de testes e pipeline que a Saga funciona de verdade?
- **Plano de Testes:** Descreva como você estruturaria o teste automatizado para simular a rejeição na reserva de estoque após a cobrança e validar que a compensação automática transiciona o pedido para cancelado com sucesso.
  > **Teste de integração da saga** (roda no pipeline a cada PR):
  > - **Ambiente:** Testcontainers sobe PostgreSQL e Kafka reais; WireMock simula o gateway de pagamento (cobrança aprovada e endpoint de estorno).
  > - **Cenário principal:**
  >   1. o Estoque (stub consumidor) responde `EstoqueReservado` na reserva e `EstoqueRejeitado` na confirmação;
  >   2. o teste dispara o checkout;
  >   3. com Awaitility, espera o pedido chegar a `CANCELADO_COM_ESTORNO`;
  >   4. verifica no WireMock que o estorno foi chamado **exatamente uma vez**, com a Idempotency-Key `estorno-{pedidoId}`.
  > - **Variações:**
  >   - o estorno falha nas 2 primeiras tentativas (Scenarios do WireMock) e mesmo assim termina estornado, sempre com a mesma chave;
  >   - a mensagem `EstoqueRejeitado` é entregue em duplicidade e só um estorno acontece;
  >   - o broker cai no meio do fluxo e as mensagens da outbox são publicadas quando ele volta.
  > - **Complementos:** testes de contrato das mensagens (Spring Cloud Contract / Pact) entre os serviços e injeção de falhas com Toxiproxy em homologação.
