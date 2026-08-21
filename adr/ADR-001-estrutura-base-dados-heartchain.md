# ADR-001: Estrutura Base de Dados de Contas e Administração de Grupos — HeartChain

## Status

Esperando Aceite( David - ✅, Guilherme - )

## Contexto

O HeartChain é uma plataforma de rede de apoio familiar, cujo objetivo é organizar e facilitar a divisão de cuidados entre responsáveis e pessoas de confiança (rede de apoio), em torno de uma ou mais pessoas cuidadas — que podem ser crianças/bebês, pessoas PCD, idosos ou pets.

O app não nasceu com escopo fechado em "cuidado de bebês": ao longo do desenho de funcionalidades, o conceito evoluiu para cobrir qualquer relação de cuidado dentro de uma família, incluindo cenários como:

- Uma criança cuidada por pais, avós, tios e amigos autorizados
- Um idoso apoiado pelos próprios filhos
- Uma pessoa PCD (criança ou adulta) com rede de apoio própria
- Um pet cuidado pela família

Um mesmo indivíduo (Account) pode assumir papéis diferentes dependendo do contexto — por exemplo, ser **responsável** pelos próprios filhos e, ao mesmo tempo, **rede de apoio** de um idoso (ex: ajudar a cuidar do próprio pai, cujo grupo é administrado por um irmão).

Era necessário decidir a estrutura de dados que sustenta:

1. Contas de usuários (responsáveis e rede de apoio)
2. Como essas contas se organizam em torno de uma pessoa cuidada
3. Como permissões e papéis são atribuídos
4. Onde termina a responsabilidade do sistema e onde começam questões que são de foro pessoal/familiar, fora do escopo do app

## Decisão

### 1. Conta unificada (`Account`)

Optamos por **uma única tabela de conta** (`Account`), em vez de tabelas separadas para "Responsável" e "Rede de Apoio".

**Motivo:** o papel de uma pessoa (responsável ou rede de apoio) não é uma característica fixa da pessoa, e sim da sua participação em um grupo específico. A mesma pessoa pode ser responsável em um grupo e rede de apoio em outro. Manter contas separadas geraria duplicação de estrutura (os campos eram praticamente idênticos: email, senha, CPF, nome, telefone, foto de perfil) e complicaria a autenticação, que passaria a precisar checar duas tabelas para saber se um email pertence a algum usuário.

Campos da `Account`:

- `id` (PK)
- `email`
- `senha_hash`
- `cpf`
- `nome`
- `telefone`
- `foto_perfil` (opcional)
- `created_at`

### 2. `Grupo` como unidade central de organização

O `Grupo` é a entidade que amarra tudo: pessoa(s) cuidada(s), responsáveis e rede de apoio. Cada grupo tem um único administrador, referenciado diretamente por FK:

- `id` (PK)
- `nome_grupo` (opcional)
- `admin_account_id` (FK → `Account.id`)
- `created_at`

**Decisão de projeto:** `admin_account_id` é a **única fonte de verdade** sobre quem administra o grupo — não existe uma segunda forma de marcar "admin" em nenhuma tabela de junção. Isso foi uma correção deliberada de um desenho anterior, em que "admin" também podia ser marcado como valor de permissão dentro da tabela de junção `GrupoResponsavel`, criando risco de dois lugares divergentes dizendo coisas diferentes sobre quem é o admin.

### 3. Papéis resolvidos via tabelas de junção (N:N)

Como um `Account` pode participar de múltiplos grupos com papéis diferentes, e um `Grupo` pode ter múltiplos responsáveis e múltiplos membros de rede de apoio, os relacionamentos são N:N, resolvidos por duas tabelas de junção:

**`GrupoResponsavel`** (papel: responsável)

- `id` (PK)
- `grupo_id` (FK → `Grupo.id`)
- `account_id` (FK → `Account.id`)
- `nivel_acesso_no_grupo`
- `tipo_afinidade` (pai / mãe / tio / cuidador / etc. — valores a definir)
- `created_at`

**`GrupoRedeApoio`** (papel: rede de apoio)

- `id` (PK)
- `grupo_id` (FK → `Grupo.id`)
- `account_id` (FK → `Account.id`)
- `nivel_acesso_no_grupo`
- `created_at`

Isso resolve diretamente o cenário motivador: a mesma `Account` pode ter um registro em `GrupoResponsavel` para o grupo dos próprios filhos e, separadamente, um registro em `GrupoRedeApoio` para o grupo de um idoso da família — sem duplicar conta e sem ambiguidade de papel.

#### 3.1. Múltipla participação da mesma Account em grupos diferentes

A única restrição que o modelo impõe é: **um grupo tem exatamente um admin** (garantido pela FK `admin_account_id` em `Grupo`). Não existe nenhuma restrição sobre quantos grupos uma `Account` pode administrar, nem sobre quantos grupos pode participar como responsável ou rede de apoio. Isso permite, sem qualquer alteração no modelo, cenários de múltiplas camadas de cuidado para a mesma pessoa, por exemplo:

- **Ser admin de um grupo e responsável (não-admin) de outro:** uma `Account` pode ser `admin_account_id` do Grupo A (ex: seus filhos) e, ao mesmo tempo, ter um registro em `GrupoResponsavel` vinculado ao Grupo B (ex: um idoso da família, cujo grupo foi criado e administrado por outro parente) — nesse segundo grupo, seu `nivel_acesso_no_grupo` é definido pelo admin daquele grupo, não por você.
- **Ser admin de mais de um grupo:** nada impede que `Grupo.admin_account_id` aponte pra mesma `Account` em dois ou mais grupos distintos (ex: admin do grupo dos filhos e admin do grupo de um pet, ambos criados pela mesma pessoa).
- **Combinar os três papéis simultaneamente:** admin de um grupo, responsável não-admin de outro, e rede de apoio de um terceiro — tudo isso são apenas linhas independentes nas tabelas de junção, todas referenciando o mesmo `Account.id`.

### 4. `PessoaCuidada` pertence a exatamente um `Grupo`

- `id` (PK)
- `grupo_id` (FK → `Grupo.id`)
- `nome`
- `idade` / `data_nascimento`
- `tipo` (crianca / idoso / pcd / pet)
- `descricao`
- `documento_opcional` \* (texto livre, não validado — sugestão, sem obrigatoriedade)
- `created_at`

**Decisão deliberada:** a pessoa cuidada **não exige comprovação documental**. O HeartChain não se propõe a ser um sistema de verificação de identidade — é uma ferramenta de apoio à organização da agenda e da rede de cuidado. Os dados cadastrados são tratados como verdade dada pelo responsável que os inseriu, sem checagem cruzada com nenhuma fonte externa.

### 5. Escopo do que o app resolve vs. o que fica de fora

Esta foi a decisão mais importante desta ADR, e a que travou o desenho por um tempo até ser explicitada:

**O app não arbitra conflitos entre pessoas.** Cenários como divórcio, desavenças entre responsáveis, ou discordância sobre quem deveria fazer parte da rede de apoio **não são tratados pela aplicação**. A resolução de qualquer conflito desse tipo é responsabilidade das pessoas envolvidas, fora do sistema.

Dentro do app, isso se traduz em uma regra simples e sem exceções:

- O **admin do grupo** (quem criou o grupo) tem autoridade para remover outros responsáveis ou membros da rede de apoio daquele grupo, sem necessidade de justificar ou mediar a decisão pelo sistema.
- Se um responsável removido (ou insatisfeito com uma decisão do admin) quiser continuar participando da vida da pessoa cuidada dentro do app, a saída é **criar seu próprio grupo** e cadastrar (ou vincular) a pessoa cuidada de forma independente.

Essa decisão é o que sustenta, arquiteturalmente, o cenário do casal divorciado: cada um pode ter sua própria conta, seu próprio grupo, e cadastrar separadamente o mesmo filho — sem qualquer vínculo entre os dois registros. O sistema não tenta reconciliar, verificar ou impedir isso.

## Consequências

**Positivas:**

- Modelo de dados simples e direto: 5 tabelas cobrem toda a base de contas, grupos e pessoas cuidadas.
- Autenticação centralizada em uma única tabela (`Account`), independentemente do papel exercido em cada grupo.
- Suporte nativo a cenários de múltiplos papéis (mesma pessoa como responsável em um grupo e rede de apoio em outro) sem gambiarra.
- O sistema fica deliberadamente fora de disputas humanas que não lhe cabem resolver, reduzindo complexidade de produto e de moderação.

**Negativas / trade-offs aceitos:**

- **Duplicação intencional de pessoa cuidada:** a mesma pessoa real (ex: um filho de pais separados) pode existir como dois registros completamente independentes em `PessoaCuidada`, em dois grupos diferentes, sem nenhum vínculo entre eles. Pedidos de ajuda, agenda e rede de apoio de um grupo são invisíveis para o outro grupo, mesmo tratando-se da mesma criança na vida real.
- **Sem verificação de identidade:** como não há checagem documental, o sistema depende inteiramente da boa-fé de quem cadastra. Isso é aceitável dado o modelo de confiança do produto (a rede de apoio é convidada, presumindo relação de confiança prévia), mas é uma limitação consciente, não uma omissão.
- **Sem histórico de decisões do admin:** a remoção de um responsável ou membro da rede de apoio pelo admin não tem, neste desenho, um registro de auditoria/histórico. Isso pode ser revisitado se o produto crescer e precisar de rastreabilidade dessas ações.

## Alternativas consideradas

1. **Manter tabelas de conta separadas (Responsável e Rede de Apoio):** descartada por gerar duplicação de estrutura e dificultar autenticação, além de não suportar naturalmente o cenário de uma mesma pessoa exercendo os dois papéis em grupos diferentes.
2. **Permitir múltiplos admins por grupo (via tabela de junção, sem FK direta em `Grupo`):** descartada em favor de uma única FK (`admin_account_id`), por garantir por schema que existe exatamente um admin por grupo, evitando inconsistência de aplicação.
3. **Exigir verificação documental da pessoa cuidada:** descartada — foge do propósito do app (ferramenta de organização de agenda/rede de apoio, não sistema de identidade verificada).
4. **Sistema de mediação/arbitragem de conflitos entre responsáveis dentro do app:** descartada — considerada fora do escopo e da responsabilidade do produto; a solução adotada (admin decide, quem discordar cria seu próprio grupo) mantém o sistema simples e neutro.

## Questões em aberto para decisões futuras

- **Deduplicação de pessoa cuidada entre grupos:** ainda não decidido se, no futuro, valerá a pena impedir ou sinalizar quando a "mesma" pessoa cuidada é cadastrada em mais de um grupo (para evitar sobrecarga de base de dados ou dados divergentes). Essa regra, se implementada, precisaria considerar que **não deve afetar o cadastro de pets**, já que não há motivo de negócio para restringir isso no caso de animais. Decisão adiada — mantém-se, por ora, sem qualquer restrição de duplicação.
- **Histórico/auditoria de remoções feitas pelo admin do grupo:** avaliar, mais à frente, se será necessário registrar essas ações para fins de transparência ou suporte.
- **Valores possíveis de `nivel_acesso_no_grupo` e `tipo_afinidade`:** ainda a definir a enumeração final desses campos.
