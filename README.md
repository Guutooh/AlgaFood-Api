# AlgaFood — API REST de Delivery de Comida

API REST completa para um sistema de delivery, com gestão de restaurantes, cardápios, pedidos e controle de acesso. O projeto vai além do CRUD básico: o modelo de domínio tem regras de negócio próprias, o fluxo de pedidos é controlado por transições de status, o acesso é protegido por OAuth2 com Authorization Server embutido, e o sistema emite e-mails e relatórios PDF de forma nativa.

---

## O que o sistema faz

Restaurantes têm cozinha, endereço, taxa de frete, formas de pagamento aceitas e um cardápio de produtos com foto. Pedidos são criados com itens desse cardápio, endereço de entrega e forma de pagamento — e passam por um fluxo de status: `CRIADO → CONFIRMADO → ENTREGUE` ou `CRIADO → CANCELADO`. Cada transição é validada pela própria entidade: um pedido entregue não pode ser cancelado, por exemplo.

Quando um pedido é confirmado ou cancelado, o sistema dispara um evento de domínio que é escutado por um listener responsável por enviar o e-mail de notificação ao cliente.

- CRUD completo de restaurantes, cozinhas, cidades, estados, formas de pagamento, grupos e permissões
- Ativação/inativação de restaurantes e abertura/fechamento para pedidos
- Gestão de responsáveis pelo restaurante (usuários vinculados a ele)
- Cardápio com ativação de produtos e upload de foto (local ou S3)
- Emissão de pedidos com cálculo automático de subtotal, frete e valor total
- Fluxo de status com validação de transições na camada de domínio
- Notificação de e-mail ao confirmar ou cancelar pedido (SMTP, sandbox ou fake para desenvolvimento)
- Estatísticas de vendas diárias com filtro por restaurante e intervalo de datas
- Exportação do relatório de vendas em PDF via JasperReports
- API versionada em `/v1/` com suporte a HATEOAS — respostas incluem links para ações disponíveis
- Documentação Swagger UI integrada ao fluxo OAuth2 com PKCE

---

## Tecnologias

**Back-end**

- Java 17 + Spring Boot 3
- Spring Authorization Server — Authorization Server embutido na própria API, sem servidor externo; clientes e autorizações persistidos no MySQL via JDBC
- Spring Security Resource Server — valida tokens JWT consultando o JWKS endpoint da própria aplicação
- Spring Data JPA + Hibernate — persistência com MySQL; repositórios customizados com Criteria API e Specifications para filtros dinâmicos
- Spring Session + Redis — armazenamento de sessão do Authorization Server
- ModelMapper — conversão entre entidades e modelos de resposta
- Bean Validation com validações customizadas: `@ValorZeroIncluiDescricao` (regra de negócio cross-field), `@FileSize` e `@FileContentType` (validação de upload)
- JasperReports — compilação e exportação de relatório em PDF
- FreeMarker — templates HTML para e-mails transacionais
- AWS SDK S3 — armazenamento de fotos de produtos em bucket S3 com acesso público
- Springdoc OpenAPI 2 — documentação com Swagger UI configurada para o fluxo Authorization Code com PKCE
- Flyway — versionamento e migração do schema do banco de dados
- Lombok — redução de boilerplate
- Cache HTTP — uso de `ETag` e `shallow ETags` para otimizar respostas que não mudaram

**Testes**

- Testes de integração com Spring Boot Test + REST Assured — cobrindo cenários de criação, validação e regras de negócio nos principais endpoints

**Infraestrutura**

- MySQL 8.0 — banco principal; também persiste clientes OAuth2, autorizações e consentimentos
- Redis — sessão do Authorization Server
- Nginx — proxy reverso na frente da API
- Docker e Docker Compose — ambiente completo containerizado
- Amazon Web Services (AWS) — deploy em produção com EC2, RDS (MySQL) e S3

---

## Como a autenticação funciona

O Authorization Server está embutido na própria aplicação — não há Keycloak ou servidor separado. Os clientes OAuth2 (`algafood-web`, `algafood-backend`) são registrados no banco via `JdbcRegisteredClientRepository`.

O fluxo padrão é Authorization Code com PKCE. O usuário é redirecionado para `/login`, autentica com e-mail e senha (carregados do banco via `JpaUserDetailsService`), e aprova os escopos na tela de consentimento em `/oauth2/consent`. O Authorization Server emite um token JWT assinado com chave RSA carregada de um keystore JKS.

O token é customizado pelo `OAuth2TokenCustomizer`: além dos escopos padrão, ele injeta o `usuario_id` e a lista de `authorities` (permissões do usuário extraídas dos grupos) diretamente no payload do JWT.

No Resource Server, o `JwtAuthenticationConverter` extrai tanto os escopos quanto essas `authorities` do token e os carrega no contexto de segurança. O controle de acesso nos endpoints é feito pela anotação customizada `@CheckSecurity`, que encapsula as regras via `@PreAuthorize` e `@PostAuthorize`. Exemplos reais do projeto:

- `@CheckSecurity.Pedidos.PodeBuscar` — verifica se tem `SCOPE_READ` e, no retorno, se o usuário é o próprio cliente, o responsável pelo restaurante ou tem a permissão `CONSULTAR_PEDIDOS`
- `@CheckSecurity.Restaurantes.PodeGerenciarFuncionamento` — verifica se o usuário autenticado é responsável pelo restaurante específico que está tentando abrir/fechar
- `@CheckSecurity.UsuariosGruposPermissoes.PodeAlterarPropriaSenha` — permite apenas que o próprio usuário altere sua senha

---

## Como rodar o projeto

Você vai precisar de Java 17+, Docker e Docker Compose instalados.

**1. Sobe os serviços com Docker Compose**

```bash
docker-compose up -d algafood-mysql algafood-redis
```

Isso sobe o MySQL na porta 13306 (sem senha de root) e o Redis. O banco `algafood` é criado automaticamente.

**2. Inicia a API**

```bash
git clone https://github.com/seu-usuario/algafood-api.git
cd algafood-api
./mvnw spring-boot:run
```

A API sobe em `http://localhost:8080`. A documentação com Swagger UI fica em `http://localhost:8080/swagger-ui.html`.

> Para rodar com nginx na frente, build a imagem com `docker build -t algafood-api .` e suba o `docker-compose` completo. O Nginx expõe a aplicação na porta 80.

**Portas**

| Serviço         | Porta  |
|-----------------|--------|
| API / Auth Server | 8080 |
| MySQL           | 13306  |
| Redis           | 6379   |
| Nginx           | 80     |

**Endpoints do Authorization Server**

| Endpoint              | URL                                        |
|-----------------------|--------------------------------------------|
| Authorization         | http://localhost:8080/oauth2/authorize     |
| Token                 | http://localhost:8080/oauth2/token         |
| JWKS                  | http://localhost:8080/oauth2/jwks          |
| Consentimento         | http://localhost:8080/oauth2/consent       |
| Clientes autorizados  | http://localhost:8080/oauth2/authorized-clients |

---


---

## Modelo de domínio

Estas são as entidades que sustentam o sistema e o papel que cada uma desempenha:

**Restaurante** — entidade central do sistema. Tem nome, taxa de frete, endereço e dois estados independentes: `ativo/inativo` (controle administrativo, gerenciado por admins) e `aberto/fechado` (controle operacional, gerenciado pelos responsáveis). Um restaurante só pode abrir se estiver ativo. Ele se relaciona com uma cozinha, com um conjunto de formas de pagamento aceitas, com seus responsáveis (usuários) e com seu cardápio de produtos.

**Cozinha** — classifica o tipo culinário do restaurante (ex: Brasileira, Italiana, Japonesa). Cada restaurante obrigatoriamente pertence a uma cozinha. É usada também como filtro na listagem de restaurantes.

**Cidade** — representa uma cidade onde um restaurante ou um endereço de entrega está localizado. Toda cidade pertence a um estado. O endereço dos restaurantes e o endereço de entrega dos pedidos referenciam uma cidade.

**Estado** — representa uma unidade federativa. Serve como agrupador de cidades e é usado na composição de endereços. Dado praticamente estático, mas gerenciável via API.

**FormaPagamento** — define os meios de pagamento disponíveis no sistema (ex: Cartão de Crédito, PIX, Dinheiro). Cada restaurante configura quais formas aceita. Na emissão de um pedido, a forma de pagamento escolhida é validada contra as formas aceitas pelo restaurante.

**Grupo** — agrupa um conjunto de permissões e é atribuído a usuários. Funciona como um papel (role) composto: em vez de atribuir permissões uma a uma, o admin atribui um grupo ao usuário. Um usuário pode pertencer a múltiplos grupos, e suas permissões efetivas são a união das permissões de todos os seus grupos.

**Produto** — representa um item do cardápio de um restaurante. Tem nome, descrição, preço e pode ser ativado ou inativado sem ser removido. Cada produto pode ter uma foto associada (`FotoProduto`), armazenada localmente ou no S3 — a troca de foto remove automaticamente a anterior do storage.

**Usuário** — representa qualquer pessoa que interage com o sistema, seja como cliente (quem faz pedidos) ou como responsável por um restaurante (quem gerencia cardápio, abre e fecha o restaurante). Um usuário pertence a um ou mais grupos, e suas permissões efetivas são derivadas desses grupos. A senha é armazenada com hash BCrypt.

**Permissão** — representa uma capacidade específica dentro do sistema (ex: `EDITAR_RESTAURANTES`, `CONSULTAR_PEDIDOS`, `GERAR_RELATORIOS`). É o nível mais granular do controle de acesso. As permissões são carregadas no token JWT pelo `OAuth2TokenCustomizer` e verificadas em cada endpoint via `@CheckSecurity`.

---

## Estrutura do projeto

```
api/
├── v1/controller/          → endpoints REST versionados
├── v1/model/               → modelos de resposta com links HATEOAS
├── v1/model/input/         → modelos de entrada validados
├── v1/assembler/           → conversão entidade ↔ modelo via ModelMapper
├── v1/openapi/             → interfaces com anotações OpenAPI separadas dos controllers
├── exceptionhandler/       → handler global com respostas no padrão RFC 7807 (Problem Details)
core/
├── security/               → Resource Server config, CORS, @CheckSecurity, AlgaSecurity
├── security/authorizationserver/ → Authorization Server, JpaUserDetailsService, JWT customizer
├── email/                  → configuração e seleção de implementação de e-mail
├── storage/                → configuração e seleção de implementação de storage
├── springdoc/              → config do Swagger com suporte a paginação e OAuth2
├── validation/             → validações customizadas (@ValorZeroIncluiDescricao, @FileSize, @FileContentType)
domain/
├── model/                  → entidades JPA com regras de negócio (ex: transições de status no Pedido)
├── repository/             → interfaces JPA + queries com Criteria API
├── service/                → serviços de domínio (emissão de pedido, fluxo, catálogo de fotos, e-mail)
├── event/                  → eventos de domínio (PedidoConfirmadoEvent, PedidoCanceladoEvent)
├── listener/               → listeners que disparam e-mails ao receber eventos de pedido
infrastructure/
├── repository/             → implementações customizadas com Specifications e Criteria API
├── service/email/          → SmtpEnvioEmailService, SandboxEnvioEmailService, FakeEnvioEmailService
├── service/storage/        → LocalFotoStorageService, S3FotoStorageService
├── service/report/         → PdfVendaReportService com JasperReports
```

---

## Principais endpoints

| Método | Rota                                          | Permissão                        | Descrição                            |
|--------|-----------------------------------------------|----------------------------------|--------------------------------------|
| GET    | /v1/restaurantes                              | autenticado                      | Lista restaurantes                   |
| POST   | /v1/restaurantes                              | EDITAR_RESTAURANTES              | Cadastra restaurante                 |
| PUT    | /v1/restaurantes/{id}/ativo                   | EDITAR_RESTAURANTES              | Ativa restaurante                    |
| PUT    | /v1/restaurantes/{id}/abertura                | responsável pelo restaurante     | Abre restaurante para pedidos        |
| GET    | /v1/restaurantes/{id}/produtos                | autenticado                      | Lista produtos do cardápio           |
| PUT    | /v1/restaurantes/{id}/produtos/{pid}/foto     | responsável pelo restaurante     | Upload de foto do produto            |
| GET    | /v1/pedidos                                   | CONSULTAR_PEDIDOS ou próprio     | Lista pedidos com filtros            |
| POST   | /v1/pedidos                                   | autenticado                      | Emite pedido                         |
| PUT    | /v1/pedidos/{codigo}/confirmacao              | gerencia pedidos do restaurante  | Confirma pedido                      |
| PUT    | /v1/pedidos/{codigo}/entrega                  | gerencia pedidos do restaurante  | Registra entrega                     |
| PUT    | /v1/pedidos/{codigo}/cancelamento             | gerencia pedidos do restaurante  | Cancela pedido                       |
| GET    | /v1/estatisticas/vendas-diarias               | GERAR_RELATORIOS                 | Dados de vendas diárias              |
| GET    | /v1/estatisticas/vendas-diarias?format=pdf    | GERAR_RELATORIOS                 | Relatório de vendas em PDF           |
| GET    | /v1/usuarios/{id}/grupos                      | CONSULTAR_USUARIOS               | Grupos de um usuário                 |
| GET    | /v1/grupos/{id}/permissoes                    | CONSULTAR_USUARIOS               | Permissões de um grupo               |

---

## Referência

Projeto desenvolvido seguindo o curso Especialista Spring REST da AlgaWorks.

O foco foi aprender na prática como construir uma API REST profissional — desde modelagem de domínio e tratamento de erros até segurança com OAuth2 Authorization Server, relatórios e deploy com Docker.
