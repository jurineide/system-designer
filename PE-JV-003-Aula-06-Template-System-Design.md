# Template de System Design: Aula 06 - Observabilidade & Diagnóstico em Produção

> **Contexto do Cenário: O Checkout que Trava sem Dar Erro**
>
> Sua equipe responde pelo serviço de **Checkout** do marketplace, que chama **Pagamento** e **Estoque**. Os três rodam em imagens *Distroless*.
>
> 1. **O Incidente:** Na abertura da Black Friday, com **5.000 TPS**, uma versão nova do Checkout entrou em *Canary*. Minutos depois, os painéis continuavam verdes, com latência média de 120 ms, CPU abaixo de 40% e nenhum pico de HTTP 500.
> 2. **O Impacto:** A conversão despencou. Milhares de reclamações chegaram ao suporte, todas com a mesma cena. O cliente clica em "Finalizar Pedido", a tela fica carregando e o aplicativo desiste por timeout.
> 3. **As Restrições:** A imagem não tem `sh` e não há SSH nos servidores. Cada serviço escreve logs em texto livre, do seu jeito, sem um identificador que ligue a chamada do Checkout à chamada que ele fez ao Pagamento. E dado de cliente que vai parar em log continua sujeito à LGPD.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Logs e Correlação

- **Campos do Log:** Que campos todo log do Checkout precisa ter para você encontrar, no meio de milhares de linhas, as da requisição que travou? Liste os campos e diga para que serve cada um.
  > Logs **estruturados em JSON** (um objeto por linha), para a ferramenta de logs filtrar por campo em vez de buscar texto:
  > - `timestamp` (ISO-8601, UTC): ordenar os eventos e cruzar com métricas.
  > - `level`: separar erros e avisos do resto.
  > - `service` e `version` (tag/digest da imagem): saber se a linha veio do **canary** ou da versão estável, essencial neste incidente.
  > - `pod` / `instance`: achar uma réplica com problema.
  > - `trace_id` e `span_id`: juntar todas as linhas de uma requisição, nos três serviços.
  > - `order_id`: achar a requisição a partir da reclamação do cliente.
  > - `http.route`, `http.method`, `http.status` e `duration_ms`: saber qual endpoint, com qual resultado e quanto tempo levou.
  > - `dependency` e `dependency_duration_ms`: mostrar qual chamada externa (Pagamento, Estoque, banco) segurou o tempo.
  > - `saga_step`: em qual etapa da Saga o pedido parou.
  > - `error.type` e `message`: o que aconteceu.
- **O Caminho do Contexto:** Como o mesmo identificador de rastreamento passa do Checkout para o Pagamento e o Estoque numa chamada HTTP?
  > Pelo header padrão **W3C Trace Context**, `traceparent: 00-<trace_id>-<span_id>-01`, criado pelo Checkout quando a requisição chega.
  > - O **OpenTelemetry** (agente Java ou Micrometer Tracing) instrumenta o `RestClient`/`WebClient` e injeta o header automaticamente em toda chamada ao Pagamento e ao Estoque. Do outro lado, o servidor lê o header e continua o mesmo trace, criando um span filho.
  > - O `trace_id` vai para o **MDC** do SLF4J, então aparece sozinho em todo log daquela requisição, sem o desenvolvedor lembrar de colocar.
  > - Nas mensagens assíncronas da Saga (Kafka), o mesmo `traceparent` viaja nos headers da mensagem.
- **LGPD nos Logs:** Quais dados de cliente ficam fora de logs e traces? Para os identificadores que precisam aparecer, qual tratamento você aplica e quem pode ler esses registros?
  > - **Ficam fora:** CPF, nome, e-mail, telefone, endereço, chave Pix, dados de cartão (também por PCI-DSS), tokens e senhas, e os corpos de requisição e resposta inteiros.
  > - **Identificadores que precisam aparecer:**
  >   - usa-se o `customer_id` interno (pseudônimo), que sozinho não identifica a pessoa;
  >   - se for preciso correlacionar por CPF, ele aparece como hash HMAC com chave secreta, nunca em claro;
  >   - em último caso, mascarado (`***.***.123-**`).
  > - **Duas camadas de proteção:**
  >   - um mascaramento no encoder do Logback, na aplicação;
  >   - um processador de redação no OpenTelemetry Collector, que pega o que escapar da primeira.
  > - **Acesso:** restrito por papel (RBAC) ao time do serviço e ao suporte de plantão, com auditoria de quem consultou o quê. Logs criptografados em repouso e com retenção limitada (ex.: 30 dias).

---

## 2. Métricas

- **Visão do Cliente (RED):** Quais métricas do Checkout entram no painel? Explique como você mede a duração para que o travamento de uma parte dos clientes não desapareça na média.
  > - **Rate:** requisições por segundo do `POST /checkout`, por status e por versão.
  > - **Errors:** HTTP 5xx **e também** timeouts e requisições abandonadas pelo cliente (conexão fechada antes da resposta). Neste incidente não houve 500: o erro estava escondido nos abandonos.
  > - **Duration:** medida com **histograma** (Micrometer com `publishPercentileHistogram`), e o painel mostra **p50, p95, p99 e p99,9**, nunca só a média. Se 3% dos clientes travam por 30 s e o resto responde em 80 ms, a média fica perto de 120 ms e parece saudável; o p99 mostra os 30 s na hora.
  > - **SLO e negócio:** percentual de checkouts respondidos em menos de 1 s (com alerta por *burn rate*) e pedidos concluídos por minuto. A queda de conversão aparece aqui mesmo quando a técnica parece verde.
  > - Todos os painéis separam **canary e estável** lado a lado.
- **Visão dos Recursos (USE):** Que recursos você monitora? Que sinal acusaria saturação antes do primeiro HTTP 500?
  > Para cada recurso, medimos **utilização, saturação (fila) e erros**:
  > - **Pool de conexões do banco (HikariCP):** conexões ativas, **pendentes** e tempo para obter uma conexão.
  > - **Pool HTTP do cliente para Pagamento e Estoque:** conexões em uso e requisições esperando conexão.
  > - **Threads da aplicação:** threads ocupadas e fila de requisições do Tomcat (ou virtual threads presas).
  > - **Resiliência:** estado do Circuit Breaker, vagas livres no Bulkhead e timeouts do TimeLimiter.
  > - **JVM e container:** heap, pausas de GC e *CPU throttling* do container (que fica escondido atrás de um "CPU abaixo de 40%").
  > - **Banco e mensageria:** conexões e esperas por lock no PostgreSQL, e lag do consumidor Kafka.
  >
  > **Sinal antecipado:** `hikaricp_connections_pending` acima de zero e subindo, junto com o tempo para obter conexão (ou fila no pool HTTP). As requisições ficam esperando um recurso, a CPU fica baixa e nenhum 500 aparece ainda. É o retrato deste incidente.
- **Labels:** Quais labels você permite nessas métricas e quais proíbe? Diga o efeito de cada escolha no servidor de métricas.
  > - **Permitidas (baixa cardinalidade, poucos valores possíveis):** `service`, `version`, `env`, `region`, `pod`, `http.route` como template (`/orders/{id}`, nunca a URL com o ID), `method`, `status` ou classe de status (`2xx`, `5xx`), `outcome` e `dependency`. Cada combinação vira uma série temporal, e com poucos valores o total de séries fica controlado.
  > - **Proibidas (alta cardinalidade):** `customer_id`, `order_id`, `trace_id`, e-mail, URL completa e mensagem de erro. Cada valor novo cria uma **série nova**. A 5.000 TPS seriam milhões de séries: o Prometheus estoura memória e disco, as consultas ficam lentas e o custo dispara (além do risco de LGPD).
  > - Esses detalhes vão para **logs e traces**. Para ligar os mundos, a métrica de latência carrega **exemplars**: um ponto lento no gráfico do p99 leva direto ao `trace_id` de uma requisição real.

---

## 3. Traces e Amostragem

- **A Escolha:** ( ) Head Sampling  ( ) Tail Sampling  (X) Combinação das duas
- **Justificativa:** Guardar 100% dos traces a 5.000 TPS custa caro. Qual é a sua regra de retenção, e como ela evita que justamente as requisições que travaram sejam descartadas?
  > - **Head sampling só onde não há valor diagnóstico:** decidido no início da requisição, no próprio serviço. Health checks e métricas: 0%. Endpoints de altíssimo volume e baixo risco (ex.: consulta de catálogo): pequena amostra. No fluxo de **checkout, 100% dos spans** seguem para o Collector, porque no início da requisição ainda não dá para saber se ela vai travar.
  > - **Tail sampling no OpenTelemetry Collector:** a decisão é tomada **depois que o trace termina**, olhando o trace inteiro. A regra de retenção:
  >   - guarda **100%** dos traces com erro, timeout ou requisição abandonada pelo cliente;
  >   - guarda **100%** dos traces mais lentos que o SLO (ex.: acima de 1 s);
  >   - guarda **100%** dos traces da versão **canary** durante o rollout;
  >   - guarda **5%** aleatórios do restante, como linha de base do comportamento normal.
  >
  > Assim, as requisições que travaram são exatamente as que ficam. O volume guardado cai para algo próximo de 5% do total.
  > - **Cuidados para funcionar:**
  >   - um exporter de balanceamento por `trace_id` manda todos os spans do mesmo trace para a mesma instância do Collector;
  >   - o tempo de espera da decisão (`decision_wait`) é maior que o timeout do cliente, para um trace travado não ser julgado antes de terminar.

---

## 4. Estratégia de Testes

- **Qual a nossa estratégia para testar isso e garantir que funciona?** Antes da próxima Black Friday, como você prova que os painéis e os alertas mostrariam este incidente?
  > Reproduzindo o incidente de propósito, num ambiente controlado, e conferindo que cada sinal aparece.
  > - **Game day em homologação:**
  >   1. carga realista com k6/Gatling;
  >   2. uma versão canary com Toxiproxy injetando 30 s de latência em uma pequena parte das chamadas ao Pagamento (ou um pool de conexões propositalmente pequeno);
  >   3. o teste passa se, em poucos minutos:
  >      - o p99 e os abandonos do canary sobem enquanto a média continua baixa (provando que o painel não depende da média);
  >      - `hikaricp_connections_pending` sobe antes de qualquer 500;
  >      - o alerta de *burn rate* do SLO dispara;
  >      - a análise do canary aborta sozinha.
  > - **Testes automatizados no pipeline:**
  >   - **Regras de alerta:** testadas com `promtool test rules`, com séries simuladas que imitam o incidente (média baixa e p99 alto) e verificação de que o alerta dispara.
  >   - **Teste de integração com os três serviços:** confere que os logs saem em JSON com os campos obrigatórios e que o mesmo `trace_id` aparece no Checkout, no Pagamento e no Estoque.
  >   - **LGPD:** um teste varre os logs gerados na suíte com regex de CPF, e-mail e cartão, e falha se encontrar algum.
  >   - **Tail sampling:** com uma requisição lenta forçada, confere que o trace dela foi guardado e que dá para chegar nele pelo exemplar do p99.
  > - **Monitoramento sintético em produção:** um robô faz um checkout de teste a cada minuto, medindo do ponto de vista do cliente. Se a tela "fica carregando", ele percebe mesmo quando o servidor acha que está tudo bem.
