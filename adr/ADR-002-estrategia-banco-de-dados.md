# ADR: Estratégia de banco de dados — Polyglot Persistence (Relacional + NoSQL)

## Status
Proposta

## Contexto

O HeartChain hoje modela seu domínio principal — contas, grupos, responsáveis, rede de apoio e pessoa cuidada — em um banco relacional (PostgreSQL). Essa parte do sistema funciona essencialmente como um **sistema de autorização**: define quem pode ver e agir sobre quem, e quem administra cada grupo.

Está planejada, para uma fase futura, uma funcionalidade de **"mini rede social"**: feed privado, timeline de marcos, postagens, fotos/vídeos da pessoa cuidada, visível para a rede de apoio autorizada. Esse tipo de dado tem características de acesso e crescimento bem diferentes do núcleo de contas/permissões.

Surgiu a dúvida: deveríamos migrar o projeto inteiro para um banco não relacional (MongoDB ou DynamoDB) para aproveitar melhor a modelagem de feed/rede social, ou manter dois bancos diferentes dentro da mesma arquitetura?

A decisão precisa considerar o Teorema CAP e o modelo PACELC, já que os dois contextos do sistema têm prioridades diferentes de consistência, disponibilidade e latência.

## Decisão

Adotar **polyglot persistence**: manter dois bancos de dados distintos, cada um responsável por um bounded context diferente do sistema, integrados via eventos assíncronos.

### 1. Banco relacional (PostgreSQL) — núcleo de autorização
Continua sendo o dono da verdade para:
- Account (login/autenticação)
- Grupo
- GrupoResponsavel / GrupoRedeApoio (junções e permissões)
- PessoaCuidada

**Justificativa:**
- Integridade referencial forte é necessária — uma FK órfã ou uma permissão duplicada aqui não é um bug cosmético, é uma falha de controle de acesso.
- As consultas são naturalmente relacionais (joins entre Account, Grupo e PessoaCuidada) e o schema muda pouco.
- Operações como criar grupo + vincular admin + convidar rede de apoio se beneficiam de transações ACID multi-tabela nativas.
- Perfil CAP desejado: **CP** — sob partição de rede, o sistema deve preferir recusar a operação a permitir acesso com base em uma permissão desatualizada.
- Perfil PACELC: **PC/EC** — em operação normal (sem partição), prioriza consistência sobre latência. Aceitável, pois dado de permissão/autorização não é sensível a alguns milissegundos extra de latência.

### 2. Banco de documentos (MongoDB ou DynamoDB) — feed / timeline / mini rede social
Será o dono de:
- Posts / atualizações do feed
- Timeline de marcos da pessoa cuidada
- Comentários, reações, mídia associada

**Justificativa:**
- Volume de escrita/leitura alto e crescente, sem necessidade de joins complexos.
- Schema evolutivo — formato de post/marco ainda não está fechado e deve mudar conforme o produto evolui.
- Tolerante a inconsistência momentânea (um post aparecer com atraso de milissegundos para alguém não compromete o sistema).
- Perfil CAP desejado: **AP** — disponibilidade do feed importa mais do que consistência instantânea entre réplicas.
- Perfil PACELC: **PA/EL** — sob partição, prioriza disponibilidade; sem partição, prioriza baixa latência de leitura do feed em vez de consistência rígida.

*(Escolha entre MongoDB vs DynamoDB fica em aberto para quando essa fase começar — a decisão de qual dos dois não impacta esta ADR, que trata apenas da separação relacional vs não relacional.)*

### 3. Integração entre os dois contextos
Como o projeto já está desenhado em microsserviços, a separação se encaixa naturalmente:
- A API de CRUD (Postgres) permanece dona da verdade sobre contas/grupos/permissões.
- Uma futura API de Feed/Social (Mongo/DynamoDB) fica isolada, dona apenas do conteúdo social.
- A ponte entre as duas é **assíncrona, via eventos** (aproveitando a mensageria já prevista no projeto): quando um grupo muda (membro entra/sai, permissão é alterada), a API de CRUD publica um evento; a API de Feed mantém uma cópia local e desnormalizada de "quem pode ver o feed de qual pessoa cuidada", atualizada de forma eventualmente consistente.
- Isso evita que o serviço de feed dependa de join síncrono contra o Postgres a cada leitura de timeline.

## Alternativas consideradas

**A. Um único banco não relacional (Mongo/DynamoDB) para todo o sistema.**
Rejeitada. Modelar autorização/permissões em documento exige recriar em código de aplicação garantias que o relacional dá de graça (integridade referencial, transações multi-entidade), aumentando risco em uma parte do sistema que é justamente a mais sensível a erro (controle de acesso a dados de uma pessoa cuidada, muitas vezes vulnerável — criança, idoso, PCD).

**B. Um único banco relacional para todo o sistema, incluindo o feed futuro.**
Não rejeitada de forma definitiva, mas desencorajada a longo prazo: feed de alto volume com schema evolutivo tende a gerar tabelas excessivamente genéricas (EAV) ou migrações constantes conforme o produto evolui, perdendo a principal vantagem do relacional (schema estável e consultas previsíveis).

## Consequências

**Positivas:**
- Cada contexto usa o banco alinhado ao seu perfil real de consistência/disponibilidade (CAP/PACELC), em vez de um único trade-off de compromisso para os dois.
- Isolamento de falha: uma instabilidade no banco do feed não compromete login/autorização, e vice-versa.
- Encaixa sem atrito na arquitetura de microsserviços e mensageria já planejada.

**Negativas / custos a assumir:**
- Duas tecnologias de banco para operar, versionar e monitorar em vez de uma.
- Necessidade de manter a cache de autorização no serviço de feed sincronizada via eventos — mais uma peça de infraestrutura (fila/tópico) e uma superfície a mais para bugs de consistência eventual.
- Curva de aprendizado adicional para quem for mexer no projeto, por exigir conhecimento nos dois paradigmas.

## Pendências para quando a fase do feed começar
- Escolher entre MongoDB e DynamoDB (infra gerenciada vs. flexibilidade de query/agregação).
- Definir o formato exato dos eventos de sincronização de permissão entre a API de CRUD e a futura API de Feed.
- Definir janela aceitável de defasagem entre uma mudança de permissão no Postgres e sua propagação para a cache de autorização do feed.
