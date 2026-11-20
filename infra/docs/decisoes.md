# Registro de Decisões e Riscos

Este documento registra as decisões, os riscos e as alterações feitas nos repositórios originais para a infraestrutura Docker do EasyEats.

## 1. Decisões principais

| # | Decisão | Motivo |
|---|---------|--------|
| 1 | `CORS_ALLOWED_ORIGINS` é obrigatória no Compose (`${CORS_ALLOWED_ORIGINS:?...}`), sem fallback. O default local fica na propriedade `cors.allowed-origins` do `application.properties`. | O Docker nunca usa lista implícita; a execução local direta (fora do Docker) mantém o comportamento anterior. |
| 2 | `DB_URL`, `DB_USER` e `DB_PASSWORD` são derivadas de `POSTGRES_*` no Compose e não constam no `.env.example`. | Fonte única de credenciais, sem divergência entre `db` e `backend`. |
| 3 | `additional_contexts` (`infra: .`) entrega `infra/nginx.conf` ao `Dockerfile.frontend` via `COPY --from=infra`. Exige Docker Compose 2.17.0 ou superior e BuildKit. | O contexto de build do frontend continua sendo `../frontend`, sem copiar arquivos para dentro do clone. |
| 4 | Plano B para o `nginx.conf`: gerá-lo dentro do `Dockerfile.frontend` (heredoc ou `printf`). Não adotado; só entra se a validação de `additional_contexts` falhar e o usuário confirmar. | Evita duplicar o conteúdo de `infra/nginx.conf` sem necessidade. |
| 5 | `README.md` na raiz do Docker_Repo; este documento em `infra/docs/decisoes.md`. Comandos `docker compose` rodam com `infra/` como diretório de trabalho. | Visibilidade do README; decisões junto da infraestrutura. |
| 6 | Testes do `CorsConfig` com JUnit 5 parametrizado, sem dependências além do `spring-boot-starter-test`. | Não adicionar jqwik nem outra biblioteca de teste. |
| 7 | Mobile (Expo) fora do Docker. | O app roda no celular ou emulador e acessa a API pela rede. |

## 2. Riscos aceitos e pontos de atenção

- **Usuário do Postgres como superusuário.** O backend usa `POSTGRES_USER`, criado na inicialização do banco, sem usuário de aplicação com privilégios reduzidos. Risco aceito em troca de uma fonte única de credenciais.
- **Default de CORS no `application.properties`.** Se `CORS_ALLOWED_ORIGINS` for removida do `environment` do serviço `backend` no Compose, o container passa a usar silenciosamente os padrões (`localhost` 3000, 4200, 5173 e 8100). Mitigações: o valor usa `${...:?}` no Compose (removê-lo exige edição deliberada) e este registro.
- **Origens da LAN manuais.** Origens como `http://192.168.x.x:4200` precisam ser listadas em `CORS_ALLOWED_ORIGINS` no `infra/.env`. Um navegador fora da lista recebe erro de CORS.
- **Actuator exposto.** `management.endpoints.web.exposure.include=*` permanece **inalterado** no `application.properties` e expõe todos os endpoints do Actuator. É um risco de exposição, aceito porque a configuração está fora do escopo desta infraestrutura.
- **Credenciais comprometidas no histórico Git.** O `jwt.secret` fixo e a senha `root` do banco já estiveram versionados no histórico Git do API_Repo (`Easy-Eats-Api-2`). O histórico não foi reescrito. Esses valores devem ser considerados **comprometidos** e **rotacionados** (gerar novo `JWT_SECRET` e usar nova senha do banco em qualquer ambiente que os tenha usado).
- **`POSTGRES_*` só valem na criação do volume.** Alterar essas variáveis depois não muda as credenciais de um banco já inicializado. `docker compose down -v` apaga o DB_Volume e causa perda irreversível de dados.

## 3. Flyway e Hibernate

Configuração mantida sem alteração: `spring.jpa.hibernate.ddl-auto=update` e todas as propriedades `spring.flyway.*` (`enabled=true`, `locations=classpath:db/migration`, `baseline-on-migrate=true`, `out-of-order=false`, `placeholders.schema=public`, `baseline-version=0`).

- **Ordem de execução.** O Flyway executa as migrations antes da inicialização do Hibernate.
- **Migrations V3 e V4.** Ambas usam `CREATE TABLE IF NOT EXISTS` (V3: `usuario`, `role`, `usuario_roles`; V4: `tb_estoques` e demais tabelas do cardápio). Os inserts da V4 usam proteção contra duplicidade (por exemplo, `WHERE NOT EXISTS`).
- **Risco de divergência de schema.** Com `ddl-auto=update`, o Hibernate pode criar ou ajustar objetos fora das migrations. O schema gerado pelo Hibernate e o schema das migrations podem divergir.
- **`ddl-auto=update` não remove nem altera colunas existentes.** Ele só adiciona tabelas e colunas ausentes. Mudanças de tipo, como `INTEGER` para `NUMERIC`, não são aplicadas por ele.
- **Migration V5 e tipos de `Estoque`.** Releitura dos arquivos reais no clone `backend/`:
  - `V4__insert_dados_iniciais_cardapio.sql` cria `tb_estoques` com `qtd_atual INTEGER NOT NULL` e `qtd_minima INTEGER NOT NULL`.
  - `V5__estoque_quantidade_decimais.sql` executa `ALTER TABLE tb_estoques ALTER COLUMN qtd_atual TYPE NUMERIC(12, 3), ALTER COLUMN qtd_minima TYPE NUMERIC(12, 3);`.
  - `Estoque.java` mapeia `qtdAtual` e `qtdMinima` como `BigDecimal` com `@Column(nullable = false, precision = 12, scale = 3)`.
  - Conclusão: as colunas eram `INTEGER` na V4 e passam a `NUMERIC(12,3)` na V5, o que coincide com `BigDecimal` com `precision = 12` e `scale = 3`. Após a V5 não há divergência de tipo. Com apenas a V4 aplicada haveria divergência, e o `ddl-auto=update` não a corrigiria.
- **Banco novo.** O Flyway cria o histórico (`flyway_schema_history`) e aplica V1 a V5 em ordem.
- **Banco existente sem `flyway_schema_history`.** Com `baseline-on-migrate=true` e `baseline-version=0`, o Flyway cria o baseline na versão 0 e aplica V1 em diante. Como V3 e V4 usam `CREATE TABLE IF NOT EXISTS`, tabelas já existentes não causam erro nelas. Migrations sem proteção equivalente, ou que dependam de um schema diferente do esperado, podem falhar.
- **Falha de migration.** Uma migration com falha interrompe a inicialização do Backend_Service. O container fica sem subir até a causa ser corrigida.
- **`out-of-order=false`.** Migrations com versão inferior à última aplicada são rejeitadas.

## 4. Registro de Mudanças (Change_Log)

Alterações feitas nos repositórios originais. Conferidas com `git diff` no clone `backend/`. Nenhuma regra de negócio foi modificada e o histórico Git dos repositórios originais não foi reescrito.

### API_Repo (`backend/`, origem `Easy-Eats-Api-2`)

| Arquivo | Alteração | Motivo |
|---------|-----------|--------|
| `src/main/resources/application.properties` | `spring.datasource.url`, `spring.datasource.username` e `spring.datasource.password` passam a ler `${DB_URL}`, `${DB_USER}` e `${DB_PASSWORD}` (antes: `jdbc:postgresql://localhost:5432/easy_eats`, `postgres` e a senha fixa). `jwt.secret` passa a ler `${JWT_SECRET}` (antes: valor Base64 fixo). Nova propriedade `cors.allowed-origins=${CORS_ALLOWED_ORIGINS:http://localhost:3000,http://localhost:4200,http://localhost:5173,http://localhost:8100}`. | Remover credenciais e segredo do arquivo versionado e permitir configuração por variáveis de ambiente. `DB_PASSWORD` e `JWT_SECRET` não têm valor padrão: a aplicação falha na inicialização sem eles. O default de CORS preserva o comportamento da execução local. |
| `src/main/java/com/easyeats/api/config/CorsConfig.java` | A lista fixa de origens foi substituída por um campo `allowedOrigins`, preenchido no construtor a partir de `@Value("${cors.allowed-origins}")`. Novo método `static List<String> parseOrigins(String raw)`: divide por vírgula, aplica `trim`, descarta itens vazios e lança `IllegalStateException` se a lista ficar vazia (ou se o valor for nulo). Imports de `Value` e `Arrays` adicionados. Métodos (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`), headers e `allowCredentials=false` não mudaram. | Origens CORS configuráveis, incluindo origens da LAN (`http://192.168.x.x:4200`), sem alterar a política de métodos, headers e credenciais. |

Arquivos do API_Repo **não** alterados: `pom.xml`, migrations (`V1` a `V5`), entidades (incluindo `Estoque.java`) e demais classes.

Pendente: o teste `backend/src/test/java/com/easyeats/api/config/CorsConfigTest.java` (tarefas opcionais 3.3 a 3.6 do plano) **não foi criado**; o arquivo não existe no clone. Se for criado, deve ser acrescentado a esta tabela.

### Frontend (`frontend/`)

Nenhuma alteração. `git -C frontend status --short` não lista modificações. `apiUrl` permanece `http://localhost:8080` e o projeto Angular não foi tocado.

### Mobile (`mobile/`)

Nenhuma alteração registrada nesta infraestrutura. O app continua usando Expo, Development Builds e `EXPO_PUBLIC_API_URL`.
