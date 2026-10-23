# DMS

## Visão geral
O projeto DMS migra a base SQL Server 2027 on-prem **SGNPRD** (schema dbo, bases db1/db2) para o **Amazon RDS for PostgreSQL Multi-AZ** (schemaapp), na conta target da AWS. Hoje a SGNPRD atende as apps legadas e o middleware-consumer. Depois da migração, a base passa a atender a app modernizada, que já roda na AWS. A cópia é feita pelo **AWS DMS** (full load + CDC) via Direct Connect, com conversão **1:1** do schema. Na virada, toggles no Portal Manager atuam junto com a troca de DNS. Uma janela de fallback com CDC reverso permite voltar ao legado.

## Caso de negócio
- Backup, alta disponibilidade e patch da base passam a ser responsabilidade da AWS, com um banco gerenciado.
- Deixa de existir licença de SQL Server.
- A base fica junto da app modernizada, que já está na AWS.
- Não há números de custo evitado nem de valor gerado na base aprovada (ver Questões em aberto).

## Objetivos de negócio
- Tirar a base SGNPRD do SQL Server on-prem e levá-la para um banco gerenciado na AWS (RDS PostgreSQL Multi-AZ).
- Eliminar a licença de SQL Server.
- Manter o RDS apenas segundos atrás do SQL Server durante a replicação contínua.
- Concluir a virada com janela de fallback de 2 a 4 semanas e, ao final dela, desligar o on-prem.
- Não há metas mensuráveis formais (KPIs, prazos, custos) definidas.

## Características
- **Origem:** SGNPRD, SQL Server 2027 on-prem no datacenter Itaú (schema dbo, db1/db2, com log de insert/update/delete).
- **Destino:** Amazon RDS for PostgreSQL Multi-AZ (schemaapp) na conta target.
- **Migração:** AWS DMS (full load + CDC e CDC reverso) via DX. O DMS roda na AWS e se conecta ao on-prem pela porta 1433.
- **Segurança:** Secrets Manager para credenciais e KMS para criptografia. Todo o tráfego on-prem ↔ AWS passa pela conta transitiva (firewall e data-security rules).
- **Monitoramento:** Amazon CloudWatch, com alarmes de atraso do CDC e do slot do CDC reverso (OldestReplicationSlotLag, TransactionLogsDiskUsage).
- **Convivência:**
  - As apps legadas acessam o RDS via DX → API Gateway → Lambda **middleware-dl-handler** (read/write).
  - O **kaas-convivencia** (Kafka as a Service) recebe eventos das apps modernizadas, e o middleware-consumer on-prem consome esses eventos.
- **Consumidor downstream:** batch-democratizacao → datamesh (sor/sot/spec).
- **Ferramentas da migração, fora do runtime:** schema-forge (passo 1, transformação do schema, com STM e agentes) e data-devops-pipeline (passo 2, migração dos dados, em 8 etapas).
- **Restrições:**
  - O DMS só copia tabela→tabela e coluna→coluna, linha a linha. Não divide tabelas, não deduplica, não faz lookup nem join.
  - As apps on-prem não acessam a AWS diretamente.
- **Volumetria:** não informada.

## Estado atual e desafios
- As apps legadas e o middleware-consumer leem e gravam direto na SGNPRD. O Portal Manager controla toggles de leitura e escrita.
- Existem **consumidores ocultos** não inventariados: jobs, relatórios, integrações, linked servers e SSIS.
- A convivência pode ser longa: várias aplicações consomem a mesma base, e a modernização delas para a AWS pode demorar.
- Diferenças de comportamento entre SQL Server e PostgreSQL: collation, maiúsculas/minúsculas, fuso horário e SQL embutido nas apps.

## Solução alvo
- **Workflow:**
  1. O time inicia a task.
  2. Full load, com as apps continuando a gravar no SQL Server.
  3. Recuperação do atraso.
  4. Replicação contínua por dias ou semanas, enquanto a nova versão da app é testada.
  5. Virada.
- **Virada:**
  - pausar a escrita on-prem via toggle;
  - conferir atraso zero;
  - ajustar as sequences com setval;
  - ligar o CDC reverso;
  - trocar o DNS;
  - ativar a stack AWS via toggle read-cloud/write-cloud.
- **Fallback (2 a 4 semanas):** o CDC reverso RDS → SGNPRD mantém o legado atualizado, e as apps legadas ficam implantadas, sem tráfego. Depois o on-prem é desligado.
- **Fluxos de exceção:**
  - 6.1: breaking glass, com correções críticas pontuais;
  - 6.2: toggle de volta para o on-prem, análise e CDC reverso.
- **Rollback:**
  - Procedimento: bloquear a escrita na AWS, gravar um registro marcador e conferir que chegou ao SQL Server, desligar a cópia, religar jobs e triggers do legado e voltar o DNS.
  - Critério: dado errado ou causa desconhecida leva a rollback. Erro de app com correção conhecida é corrigido na AWS.
- **O que muda:** o modelo legado (dbo) continua na AWS. A normalização para o modelo corporativo fica para depois.

## Decisões-chave
- **Conversão 1:1 do schema, sem modelo novo na AWS.** O motivo é simplificar a migração e evitar erros no mapeamento de chaves. A normalização (ex.: cliente_contato → pessoa + contato) vira dívida técnica, a ser feita depois na cloud (expand/contract). Por isso, a anticorruption layer e a alternativa stg→corp não se aplicam à fase 1.
- **Alternativa descartada: normalizar antes de migrar.** Levaria mais tempo e estenderia a convivência com várias apps de modernização lenta.
- **AWS DMS (full load + CDC) via DX**, com o DMS rodando na AWS.
- **Virada coordenada entre app e base**, com toggles no Portal Manager e troca de DNS.
- **Janela de fallback de 2 a 4 semanas com CDC reverso.**
- **kaas-convivencia** no TO-BE, para levar eventos das apps modernizadas ao middleware-consumer on-prem.
- **Decisões que ainda precisam de aprovação formal:**
  1. Comitê de compliance do Itaú: as apps on-prem acessam o RDS direto ou pela Lambda?
  2. Patrocinador e cliente: aceite formal da dívida técnica do 1:1.
  3. Aprovação da janela de 2 a 4 semanas e dos critérios de rollback.

## Riscos e mitigações
| Risco | Mitigação |
|---|---|
| O DMS pode não suportar SQL Server 2027 como origem (CDC) e como destino (CDC reverso) | Validar na POC do DMS |
| O DMS não migra o valor das sequences | setval no runbook da virada |
| O slot do CDC reverso retém WAL | Alarmes em OldestReplicationSlotLag e TransactionLogsDiskUsage; remover o slot no fim da janela |
| Triggers do legado disparam quando o DMS grava | Tratar no runbook da virada (desligar e religar jobs e triggers) |
| Comportamento diferente entre os bancos (collation, maiúsculas, fuso, SQL embutido) | Validação na POC (sem plano detalhado) |
| Consumidores ocultos da base | Inventário dos consumidores |
| Atualização duplicada ou inconsistente no legado (CDC reverso + eventos Kafka via middleware-consumer) | Definir as responsabilidades de cada mecanismo (em aberto) |
| Rota AWS → on-prem na porta 1433 não liberada | Validar no guardrail e na conta transitiva |

## Questões em aberto
- O compliance vai permitir acesso direto ao RDS ou o acesso será sempre pela Lambda middleware-dl-handler?
- A porta 1433 (AWS → on-prem) já está liberada no guardrail e na conta transitiva?
- Qual é o inventário dos consumidores da SGNPRD, e qual o prazo final da janela de fallback dentro dos 2 a 4 semanas?
- O DMS suporta SQL Server 2027 como origem e destino? Antes, a versão "2027" tinha sido apontada como possível erro de digitação; a entrevista a confirmou.
- Kafka: quais eventos e tópicos o kaas-convivencia transporta, quem é o dono do cluster e como ele convive com o CDC reverso sem duplicar atualizações?
- O kaas-convivencia continua depois da janela de fallback ou é desligado junto com o on-prem?
- Como fica a convivência longa com apps legadas depois que o on-prem for desligado?
- Qual é o papel da tabela map_pessoa e do STM do schema-forge (renomeações como id-cli → id-pessoa) numa fase 1:1? Cada linha do STM será feita pelo DMS ou por script?
- Faltam na base: custos e valor quantificados, KPIs mensuráveis, volumetria, cronograma e datas.

## Próximos passos
1. Inventariar os consumidores ocultos: jobs, SSIS, linked servers e relatórios.
2. Validar no guardrail a rota AWS → on-prem na porta 1433 (DX).
3. Converter o schema 1:1 com o schema-forge.
4. Fazer a POC do DMS (full load + CDC com validação), incluindo o suporte ao SQL Server 2027.
5. Escrever o runbook da virada: setval das sequences, triggers, toggles no Portal Manager e DNS.
6. Configurar os alarmes do CDC reverso no CloudWatch.
7. Levar as três decisões pendentes ao compliance, ao patrocinador e ao cliente.
