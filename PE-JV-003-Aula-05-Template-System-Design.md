# Template de System Design: Aula 05 - Containers, CI/CD & Deploy sem Downtime

> **Contexto do Cenário: O Deploy que Quebrou o Checkout e o Rollback Impossível**
>
> Sua equipe está reestruturando a engenharia de release do serviço de **Checkout** após um incidente severo na Black Friday:
>
> 1. **O Incidente:** Para subir uma otimização urgente na Saga, o pipeline de produção recompilou a branch `main`, sobrescreveu a tag `checkout:latest` e iniciou a atualização. No boot da primeira instância nova, o **Flyway** renomeou a coluna `status_pagamento` para `estado_saga`.
> 2. **O Efeito em Cadeia:** Os 80% de servidores que ainda rodavam a versão anterior passaram a falhar imediatamente (`column "status_pagamento" does not exist`). Ao tentar acionar o rollback puxando `:latest`, a equipe descobriu que a imagem anterior havia sido sobrescrita e, mesmo restaurando um backup do binário antigo, ele continuou falhando porque o banco já estava alterado.
> 3. **A Carga:** 5.000 TPS ativos com múltiplas réplicas de aplicação conectadas ao mesmo banco PostgreSQL.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Empacotamento, Superfície de Ataque e Promoção Imutável

### 1.1 Anatomia da Imagem e Redução de Superfície de Ataque

- **Construção vs Execução:** Como você estruturará o empacotamento do serviço de Checkout para produção? Especifique o que deve ser usado na etapa de compilação, o que deve (e o que não deve) entrar na imagem final de execução, com qual permissão de usuário o processo rodará e qual o trade-off da sua escolha de imagem base para a segurança e para o diagnóstico em produção.
  > **Dockerfile multi-stage:**
  > - **Estágio de build:** imagem com JDK 21 e Maven (`eclipse-temurin:21-jdk`). Compila, roda os testes e gera o `.jar` (com as camadas do Spring Boot extraídas, para aproveitar cache). Esse estágio é descartado.
  > - **Imagem final:** `gcr.io/distroless/java21-debian12:nonroot`, só com a JRE e o `.jar`.
  > - **Não entra na imagem final:** JDK, Maven, código-fonte, `~/.m2`, shell, `apt`, `curl` e segredos (que chegam por variável de ambiente ou secret manager em tempo de execução).
  > - **Usuário:** `nonroot` (UID 65532), nunca root, com sistema de arquivos somente leitura no Kubernetes.
  >
  > **Trade-off:** a Distroless reduz muito a superfície de ataque e as CVEs de sistema operacional (sem shell, um invasor não tem ferramentas para usar), mas também não dá para entrar no container para investigar. O diagnóstico em produção passa a depender de observabilidade (logs, métricas, traces, Actuator), de `kubectl debug` com um container efêmero ou da variante `:debug` da imagem, usada só fora de produção.

### 1.2 Rastreabilidade, Identidade do Artefato e Parametrização

- **Da Homologação à Produção:** Como sua esteira garante que o binário executado em produção é exatamente o mesmo que passou nos testes de homologação? Explique como as configurações mudam entre ambientes (dev, staging, prod) sem recompilar o artefato e como você identifica a versão no registry para que um rollback nunca puxe um binário sobrescrito por engano.
  > - **Build once, deploy many:** a imagem é construída **uma única vez**, no merge na `main`. Homologação e produção recebem essa mesma imagem; produção nunca recompila.
  > - **Identidade imutável:** a imagem recebe uma tag com a versão e o SHA do commit (ex.: `checkout:1.8.3-a1b2c3d`), e o deploy referencia o **digest** (`checkout@sha256:...`), que é o hash do conteúdo e não muda. O registry é configurado com **tags imutáveis**, e a tag `:latest` não é usada em deploy.
  > - **Promoção:** passar para produção é só apontar o manifesto (Helm/Kubernetes) para o digest aprovado em homologação. Assim, o binário de produção é comprovadamente o mesmo que foi testado.
  > - **Configuração por ambiente (12-Factor):** o mesmo artefato lê a configuração de fora: variáveis de ambiente, ConfigMaps, Spring Profiles e segredos do Vault ou Secrets Manager. Nada de ambiente é embutido na imagem.
  > - **Rollback:** voltar é reaplicar o digest da versão anterior, que continua no registry e não pode ser sobrescrito.

---

## 2. Estratégia de Rollout e Migração de Banco sem Downtime

### 2.1 O Modelo de Rollout em Produção

- **A Escolha:** (X) Canary Release (Liberação Gradual por Percentual)  ( ) Blue-Green Deployment (Alternância de Ambientes)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Canary Release:* Como o tráfego evoluirá sob 5.000 TPS, quais sinais definirão o avanço ou a reversão (rollback) automática e como lidar com o tempo em que duas versões diferentes da API atendem clientes simultaneamente?
  - *Se você escolheu Blue-Green Deployment:* Como você gerenciará o custo de infraestrutura duplicada no pico da Black Friday, o aquecimento das novas instâncias (conexões/cache) antes da virada de chave e o risco de virar 100% do tráfego de uma só vez?
  > O Canary limita o estrago a uma fração dos clientes e não exige dobrar a infraestrutura no pico, que seria o custo do Blue-Green.
  > - **Evolução do tráfego** (Argo Rollouts ou Flagger, com pesos no ingress/service mesh): 1% → 5% → 20% → 50% → 100%, com pausas de cerca de 10 minutos entre os passos. A 5.000 TPS, até 1% já são 50 requisições por segundo, o suficiente para ter métricas confiáveis em poucos minutos.
  > - **Sinais de avanço ou rollback automático**, sempre comparando o canary com a versão estável no mesmo período:
  >   - taxa de erros 5xx;
  >   - latência p99;
  >   - taxa de falhas e compensações da Saga;
  >   - conexões pendentes no HikariCP e esperas por lock no PostgreSQL.
  >
  >   Se algum sinal piorar além do limite, a análise aborta e o tráfego volta para a versão estável sem intervenção humana.
  > - **Duas versões ao mesmo tempo:**
  >   - o banco evolui por Expand and Contract (2.2), então as duas versões funcionam com o mesmo schema;
  >   - contratos de API e eventos só mudam de forma aditiva (campos novos opcionais, eventos versionados, consumidores que ignoram campos desconhecidos);
  >   - assim, um pedido que começou numa versão e continuou na outra não quebra.

### 2.2 Evolução de Schema Sem Downtime e Plano de Rollback

- **Passo a Passo da Migração:** Descreva em etapas sequenciais como evoluir a coluna `status_pagamento` para `estado_saga` no PostgreSQL garantindo que as instâncias da versão antiga (`v1`) e da versão nova (`v2`) operem juntas sem erros de SQL.
  > Padrão **Expand and Contract**, espalhado por várias releases. Nenhuma migração apaga ou renomeia algo que uma versão em execução (ou alvo de rollback) ainda usa.
  > 1. **Expand (release da v2):** a migração Flyway só **adiciona** `estado_saga` como coluna nullable, sem default (operação instantânea no PostgreSQL). A `v1` continua funcionando, porque ignora a coluna nova.
  > 2. **Dual write (v2):** a `v2` grava nas **duas** colunas e lê `COALESCE(estado_saga, status_pagamento)`, o que cobre as linhas que a `v1` continua gravando só na coluna antiga.
  > 3. **Backfill:** um job copia `status_pagamento` para `estado_saga` nas linhas antigas, em lotes pequenos (ex.: 5.000 linhas por vez), para não travar a tabela a 5.000 TPS.
  > 4. **Migrate (v3):** quando a `v1` já saiu de produção e o backfill terminou, a `v3` passa a ler só `estado_saga`. Se preciso, a obrigatoriedade entra com `CHECK ... NOT VALID` seguido de `VALIDATE CONSTRAINT`, sem lock longo.
  > 5. **Contract (release futura):** só quando nenhuma versão em produção nem alvo de rollback usa a coluna antiga, uma migração faz `DROP COLUMN status_pagamento`.
  >
  > Além disso, as migrações rodam num **Job separado antes do rollout**, com `lock_timeout` baixo, e não no boot de cada réplica, como aconteceu no incidente.
- **Garantia de Rollback Imediato:** Caso a versão `v2` apresente degradação grave de performance com 20% do tráfego na Black Friday, qual é o procedimento exato de rollback e por que o seu desenho de banco de dados garante que a versão `v1` voltará a assumir 100% da carga sem quebrar nem perder dados gravados pela `v2`?
  > **Procedimento:** a análise automática do canary (ou um `kubectl argo rollouts abort checkout` manual) devolve 100% do tráfego para as réplicas da `v1`, que **continuam rodando** durante todo o canary. Leva segundos e não precisa de rebuild, nem de pull de imagem nova, nem de restaurar backup. Depois, o manifesto volta a apontar para o digest da `v1` via Git (GitOps), mantendo o histórico.
  >
  > **Por que o banco não quebra:**
  > - a migração da `v2` foi só aditiva: a coluna `status_pagamento`, que a `v1` usa, continua existindo;
  > - a `v2` gravou nas duas colunas, então tudo que ela escreveu também está em `status_pagamento` e a `v1` enxerga esses pedidos;
  > - **não existe migração de "down"**: o banco fica como está, e a coluna `estado_saga` é simplesmente ignorada pela `v1` até a próxima tentativa.

---

## 3. Guardrails de Segurança, Governança e Validação da Release

### 3.1 Gates de Segurança e Guardrails no Processo de Entrega

- **Guardrails do Commit ao Deploy:** Quais portões automatizados de qualidade e segurança devem proteger a sua esteira ao longo do fluxo de desenvolvimento (desde o Pull Request até a publicação e promoção da imagem)?
  > [Sua resposta aqui]
- **Governança e Controle em Produção:** Quais *guardrails* de processo e arquitetura impedem que alterações manuais não rastreadas cheguem à produção e como equilibrar segurança rigorosa com agilidade de resposta durante incidentes críticos sem perder a trilha de auditoria?
  > [Sua resposta aqui]

### 3.2 Estratégia de Testes da Imagem e da Migração

- **Validação Pré-Tráfego:** Passar nos testes unitários durante o build do código não prova que a imagem final do container sobe nem que a migração de banco é retrocompatível.
- **Plano de Testes na Pipeline:** Descreva como você testará automaticamente a imagem construída e validará a coexistência da migração de banco de dados na esteira antes de liberar o deploy para o tráfego real de clientes.
  > [Sua resposta aqui]
