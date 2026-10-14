# Template de System Design: Aula 03 - Banco de Dados em Larga Escala

> **Contexto do Cenário: Checkout na Black Friday**
>
> Sua equipe está preparando a persistência da API de Checkout para o pico de tráfego da Black Friday:
>
> 1. **Armazenamento:** Transações financeiras de pedidos exigem garantias ACID rigorosas e consistência imediata, enquanto dados de catálogo e sessões de usuários demandam alta vazão de leitura e tolerância a partição.
> 2. **Concorrência:** A atualização de status do pedido tem baixa disputa por registro (apenas o comprador interage), mas a **reserva de estoque** sofre altíssima contenção com centenas de clientes disputando as últimas unidades do mesmo SKU promocional.
> 3. **Incidente em Produção:** Em testes de carga, o tempo de resposta p99 subiu para 4.8s e o pool HikariCP saturou devido a consultas lentas de pedidos (`WHERE customer_id = ?` e `WHERE order_status = 'processing'`).
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Adicione este bloco ao repositório contínuo do seu projeto.

---

## 1. Decisão de Armazenamento e Chave Primária (PK)

### 1.1 Motor de Banco de Dados e Paradigma de Persistência

- **A Escolha para o Checkout:** ( ) Relacional Puro (PostgreSQL) (X) Persistência Poliglota / Híbrida
- **Justificativa Arquitetural:** Avalie o Teorema CAP e o modelo de consistência (ACID vs BASE). Por que a sua escolha atende aos requisitos do Checkout sem comprometer a integridade financeira?
  > Pedidos, pagamentos e estoque ficam no **PostgreSQL** (ACID, consistência forte, lado CP do CAP): dinheiro não aceita consistência eventual. Catálogo e sessões vão para bancos **BASE / AP** (ex.: Redis, MongoDB), que escalam leitura e toleram partição; um preço de catálogo levemente desatualizado é aceitável, uma cobrança errada não. A transação financeira nunca é dividida entre bancos: os outros são sincronizados de forma assíncrona (Outbox / CDC).
- **Além do Modelo Documental (Ecossistema NoSQL):** A persistência poliglota vai muito além da dicotomia "Relacional vs MongoDB". Considerando as necessidades complementares de uma plataforma de e-commerce (como carrinhos efêmeros, busca textual de produtos, telemetria/logs, auditoria temporal ou análise de conexões/fraude), cite e comente pelo menos outros dois tipos/famílias de bancos NoSQL que poderiam coexistir na arquitetura, indicando para quais casos de uso específicos cada um seria a ferramenta ideal e por quê.
  > - **Chave-valor (Redis):** carrinho e sessão. Acesso por chave em memória (sub-milissegundo) e TTL nativo para dados efêmeros.
  > - **Motor de busca (Elasticsearch / OpenSearch):** busca textual de produtos. Índice invertido, relevância, tolerância a erro de digitação e filtros por facetas, que o SQL faz mal.
  > - **Séries temporais / colunar largo (TimescaleDB, Cassandra):** telemetria, logs e auditoria. Escrita massiva e append-only, consultas por janela de tempo.
  > - **Grafo (Neo4j):** antifraude. Encontra conexões (mesmo cartão, dispositivo ou endereço entre contas) com travessias que em SQL viram JOINs recursivos caros.

### 1.2 Estratégia da Chave Primária

- **A Escolha:** ( ) Sequence / Identity ( ) UUIDv4 (X) UUIDv7 / TSID
- **Impacto no Armazenamento:** Por que essa estratégia foi escolhida considerando inserções concorrentes e o impacto de fragmentação na árvore B-Tree (page splits)?
  > UUIDv7/TSID é **ordenado pelo tempo**: novos registros entram sempre no fim da B-Tree, sem page splits aleatórios nem fragmentação (o problema do UUIDv4, que espalha inserções pelo índice inteiro e piora o cache). Ao mesmo tempo é gerado na aplicação, sem depender de uma sequence central, funciona entre serviços e não expõe o volume de pedidos como um ID sequencial.

---

## 2. A Dualidade da Concorrência: Locks e Mitigações

Analise e defina a abordagem de concorrência para os dois fluxos críticos do negócio:

### 2.1 Mudança de Status do Pagamento (Baixa contenção por registro)

- **Abordagem de Lock:** (X) Lock Otimista (`@Version`) ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Por que essa abordagem é a mais adequada para este cenário específico?
  > Só o comprador (e o webhook do pagamento) mexe no pedido, então conflitos são raros. O lock otimista não segura lock nem conexão: só confere a versão no `UPDATE`. No caso raro de conflito (ex.: webhook duplicado), relê o pedido e decide se reaplica ou ignora.

### 2.2 Controle de Estoque Concorrente (Alta contenção no mesmo registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou (X) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Centenas de usuários disputando os últimos itens na Black Friday. Como você garante que o saldo nunca fique negativo sem derrubar o banco?
  > Com lock otimista, quase todas as tentativas falhariam e virariam uma tempestade de retries. A escolha é lock pessimista no formato mais curto possível: um **UPDATE atômico condicional**, `UPDATE estoque SET qtd = qtd - :n WHERE sku = :sku AND qtd >= :n`. O banco trava a linha só durante esse comando; se 0 linhas forem afetadas, o item esgotou. Uma constraint `CHECK (qtd >= 0)` é a última garantia. Para aliviar o banco no pico, um contador no Redis (`DECR`) pode filtrar antes, recusando pedidos quando o estoque já acabou.

### 2.3 Riscos e Mitigações

- **Análise Crítica:** Quais os riscos operacionais das abordagens de lock que você escolheu (ex: excesso de `OptimisticLockException` frustrando o cliente vs `ConnectionTimeoutException` e *deadlocks* enfileirando threads no Lock Pessimista)? Como sua arquitetura mitiga esses riscos?
  > - **Otimista:** risco de `OptimisticLockException` em excesso. Mitigação: usado só onde a contenção é baixa, com retry limitado e backoff.
  > - **Pessimista:** risco de locks longos esgotarem o HikariCP e de deadlocks. Mitigações:
  >   - transação curta: nenhuma chamada externa (como o pagamento) dentro dela;
  >   - `lock_timeout` baixo no banco e `connectionTimeout` curto no Hikari, para falhar rápido em vez de enfileirar;
  >   - pedidos com vários SKUs travam sempre na mesma ordem (por ID do SKU), o que evita deadlock;
  >   - Bulkhead e filtro no Redis limitam quantas requisições chegam a disputar a mesma linha;
  >   - reserva com expiração: se o pagamento não for confirmado em X minutos, o estoque volta.

---

## 3. Troubleshooting do Incidente e Gestão de Índices

### 3.1 Investigação de Incidente em Produção

- **Cenário:** Em testes de carga para a Black Friday, o tempo de resposta p99 do Checkout subiu repentinamente para 4.8s e o pool de conexões (HikariCP) saturou. Os logs apontam que a consulta de pedidos mais frequente (`WHERE customer_id = ? AND order_status = 'processing'`) está demorando segundos para responder.
- **Plano de Diagnóstico:** Qual é o seu processo técnico passo a passo para investigar a causa raiz dessa lentidão diretamente no banco de dados? Quais comandos, métricas ou planos você analisaria para comprovar a hipótese antes de propor qualquer mudança?
  > 1. **`pg_stat_statements`:** confirmar que essa query lidera em tempo total e médio.
  > 2. **`pg_stat_activity` e `pg_locks`:** ver se as conexões estão executando a query ou esperando lock (`wait_event`).
  > 3. **`EXPLAIN (ANALYZE, BUFFERS)`** na query: procurar `Seq Scan`, muitas linhas lidas e descartadas, e estimativa de linhas muito diferente do real.
  > 4. **`pg_stat_user_tables` / `pg_stat_user_indexes`:** `seq_scan` alto e `idx_scan` baixo na tabela de pedidos, `n_dead_tup` e último `autovacuum` / `analyze` (estatísticas velhas também geram planos ruins).
  > 5. **Métricas do HikariCP** (conexões ativas, pendentes, tempo de uso): confirmar que a saturação acompanha a lentidão da query.
  >
  > Hipótese provável: falta um índice composto, gerando Seq Scan. Valido criando o índice em homologação e repetindo o `EXPLAIN ANALYZE` e o teste de carga.

### 3.2 Decisão de Engenharia: A Abordagem de Índices

- **Dilema no Pull Request:** Para solucionar a lentidão, um desenvolvedor da equipe abriu um PR criando índices individuais para todas as colunas da tabela de pedidos que aparecem em filtros de consulta.
- **Seu Posicionamento:** Como responsável técnico, você aprovaria esse PR? Qual é a sua avaliação crítica sobre essa estratégia e quais impactos colaterais sistêmicos devem ser ponderados antes de aprovar novos índices?
  > **Não aprovaria.**
  > - Índices individuais não atendem bem o filtro combinado; e `order_status` sozinho tem baixa seletividade, quase não ajuda.
  > - Cada índice deixa **toda escrita mais cara** (INSERT/UPDATE mantêm todos os índices), justo na Black Friday, que é pico de escrita.
  > - Indexar `order_status`, que muda o tempo todo, impede os HOT updates do PostgreSQL e aumenta o trabalho do vacuum.
  > - Mais disco e mais memória disputando o cache.
  >
  > **Proposta:** um índice composto `(customer_id, order_status)` ou um índice parcial `ON pedidos (customer_id) WHERE order_status = 'processing'`, criado com `CREATE INDEX CONCURRENTLY` (sem travar a tabela), validado com `EXPLAIN ANALYZE` e acompanhando depois em `pg_stat_user_indexes` os índices que nunca são usados.
