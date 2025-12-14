# Pipeline de Integração Contínua (CI/CD)

Este projeto utiliza **GitHub Actions** para automatizar os processos de **integração contínua (CI)** e **preparação para entrega (build e Docker)**.  
Os workflows foram desenhados para validar qualidade, executar testes, gerar artefatos e preparar a aplicação para containerização de forma estruturada e escalável.

---

## Visão Geral do Pipeline

O pipeline é composto por **dois workflows principais**:

1. **Continuous Integration (`go.yml`)**
   - Executa lint, testes e build da aplicação Go.
   - Gera um artefato compilado.
2. **Docker (`Docker.yml`)**
   - Workflow reutilizável.
   - Responsável por preparar o ambiente Docker e autenticar no Docker Hub.

Fluxo geral:

push / pull request → CI → Build → Docker


Cada etapa só é executada se a anterior for concluída com sucesso.

---

## Workflow: Continuous Integration (`go.yml`)

### Quando é executado

Este workflow é acionado automaticamente quando ocorre:

- `push` na branch `main`
- `pull_request` direcionado para a branch `main`

Isso garante que qualquer alteração na branch principal ou tentativa de merge passe por validações automáticas.

---

### Job: `ci`

Responsável por **validar a qualidade e integridade do código**.

#### Etapas executadas

1. **Checkout do código**
   - Faz o download do repositório para o runner do GitHub Actions.

2. **Configuração do ambiente Go**
   - Instala e configura o Go na versão **1.22**.

3. **Inicialização do banco de dados**
   - Sobe o serviço `postgres` via `docker compose`.
   - Permite que os testes rodem em um ambiente próximo ao real.

4. **Lint**
   - Executa o `golangci-lint` para análise estática do código.
   - Garante padrões de qualidade, legibilidade e boas práticas.
   - Diretórios analisados:
     - `controllers/`
     - `database/`
     - `models/`
     - `routes/`

5. **Testes**
   - Executa os testes automatizados com `go test`.
   - As variáveis de conexão com o banco são injetadas via **GitHub Secrets**, garantindo segurança.
   - Testes executados:
     - `main_test.go`

> O job falha caso qualquer etapa de lint ou testes não seja bem-sucedida.

---

### Job: `build`

Responsável por **compilar a aplicação Go**.

#### Dependência
- Só é executado se o job `ci` finalizar com sucesso.

#### Etapas executadas

1. **Checkout do código**
2. **Build da aplicação**
   - Compila o arquivo `main.go`, gerando o binário da aplicação.
3. **Upload do artefato**
   - O binário gerado é armazenado como um **artifact** chamado `go_program`.
   - Esse artefato pode ser reutilizado em outros workflows (como Docker ou deploy).

---

### Job: `docker`

Responsável por **delegar a etapa de Docker para um workflow reutilizável**.

#### Dependência
- Só é executado se o job `build` finalizar com sucesso.

#### Funcionamento
- Utiliza o recurso `workflow_call` para chamar o workflow `Docker.yml`.
- Herda automaticamente os secrets do workflow principal.

---

## Workflow: Docker (`Docker.yml`)

Este é um **workflow reutilizável**, projetado para ser chamado por outros workflows.

---

### Quando é executado

- Não é acionado diretamente por `push` ou `pull_request`.
- É executado apenas quando chamado por outro workflow via `workflow_call`.

---

### Job: `docker`

Responsável por preparar o ambiente Docker e autenticar no Docker Hub.

#### Etapas executadas

1. **Checkout do código**
   - Necessário para acessar o Dockerfile e demais arquivos do projeto.

2. **Configuração do Docker Buildx**
   - Prepara o ambiente para builds Docker modernos e multiplataforma.

3. **Download do artefato**
   - Recupera o artifact `go_program` gerado no job `build`.
   - Permite reutilizar o binário já compilado, evitando rebuild desnecessário.

4. **Login no Docker Hub**
   - Realiza autenticação segura usando:
     - Usuário: `brunofaboci`
     - Senha armazenada em `secrets.PASSWORD_DOCKER_HUB`
   - Esse passo é essencial para permitir push de imagens Docker para o registry.

> Observação: este workflow prepara o ambiente para build e publicação de imagens Docker, podendo ser estendido futuramente para incluir `docker build` e `docker push`.

---

## Benefícios da Arquitetura do Pipeline

- **Separação clara de responsabilidades**
  - CI valida código e testes.
  - Build gera artefatos.
  - Docker lida com containerização.
- **Fail fast**
  - O pipeline falha rapidamente se lint ou testes falharem.
- **Reutilização**
  - O workflow Docker pode ser reaproveitado em outros projetos.
- **Segurança**
  - Uso de secrets do GitHub para credenciais sensíveis.
- **Escalabilidade**
  - Fácil adicionar novos jobs (deploy, segurança, etc.).

---

## Conclusão

Este pipeline implementa uma abordagem moderna de CI/CD com GitHub Actions, garantindo qualidade de código, testes automatizados, builds reprodutíveis e preparação adequada para containerização, seguindo boas práticas de engenharia de software e DevOps.
