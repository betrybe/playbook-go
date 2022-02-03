# Guia de padrões e boas práticas em Go

Versão do playbook em outros idiomas:
[Inglês](./README_EN.md)

Este documento define os padrões e boas práticas que adotamos ao escrever código Go na Trybe. Este guia está separados por tema, e priorizados pela necessidade e impacto na qualidade do nosso código.

> **Observação:** Este é um documento vivo e reflete as necessidades da Trybe, que podem mudar com o tempo, assim como as decisões que tomamos para melhor resolver nossos problemas.

## Como esse guia está organizado

- [Arquitetura](#arquitetura)
- [Variáveis de configuração](#variáveis-de-configuração)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Erros](#erros)
- [Pacotes externos](#pacotes-externos)
- [IDEs](#ides)

## Arquitetura

### Clean Architecture

Organizamos nossos códigos usando a Clean Architecture. Mais sobre a arquitetura e seus conceitos podem ser vistos nos links:

[Clean Architecture, 2 anos depois](https://eltonminetto.dev/post/2020-06-29-clean-architecture-2anos-depois/)

[Arquitetura de software e a Clean Architecture](https://speakerdeck.com/eminetto/arquitetura-de-software-e-a-clean-architecture)

[Arquitetura de software e a Clean Architecture - Hacktoberfest Brasil 2020](https://www.youtube.com/watch?v=pUl1eTTCcc4)

## Variáveis de configuração

De acordo com as [recomendações](https://12factor.net/config) documentadas no projeto [12 factor](https://12factor.net) armazenamos as configurações que mudam de acordo com o ambiente (`staging`, `dev`, `production`) em variáveis de ambiente. E as configurações que não dependem do ambiente são armazenadas em arquivos no formato [TOML](https://toml.io/en/) dentro do repositório. Para facilitar o gerenciamento das configurações usamos a biblioteca [Viper](https://github.com/spf13/viper).

## Estrutura do repositório

Usamos o conceito de monorepo, com os projetos compartilhando o mesmo repositório no Github.

**Vantagens:**

- fácil gerenciar as dependências e reaproveitar código. Com a evolução dos projetos vamos criar pacotes que são comuns a vários projetos (métricas, log, erros, etc) e ter tudo no mesmo repositório facilita a reutilização

**Desvantagens:**

- CI/CD mais complexo pois teríamos que ter opção para gerar o binário e fazer deploy de diferentes apps

Analisamos os prós e contras e chegamos a conclusão que o monorepo é a melhor opção para nossos projetos.

Abaixo um exemplo de como o repositório é organizado, com múltiplos projetos.

```
├── README.md
├── app1
│   ├── Makefile
│   ├── README.md
│   ├── api
│   │   ├── handler
│   │   │   ├── book.go
│   │   │   ├── book_test.go
│   │   │   ├── loan.go
│   │   │   ├── loan_test.go
│   │   │   ├── user.go
│   │   │   └── user_test.go
│   │   ├── main.go
│   │   ├── middleware
│   │   │   ├── cors.go
│   │   │   └── metrics.go
│   │   └── presenter
│   │       ├── book.go
│   │       └── user.go
│   ├── bin
│   │   ├── api
│   │   └── cmd
│   ├── cmd
│   │   ├── main.go
│   │   └── main_test.go
│   ├── config
│   │   ├── config.toml
│   │   ├── config_dev.toml.example
│   ├── docker-compose.yml
│   ├── entity
│   │   ├── book.go
│   │   ├── book_test.go
│   │   ├── entity.go
│   │   ├── error.go
│   │   ├── user.go
│   │   └── user_test.go
│   ├── infrastructure
│   │   └── repository
│   │       ├── book_mysql.go
│   │       └── user_mysql.go
│   └── usecase
│       ├── book
│       │   ├── inmem.go
│       │   ├── interface.go
│       │   ├── service.go
│       │   └── service_test.go
│       ├── loan
│       │   ├── interface.go
│       │   ├── service.go
│       │   └── service_test.go
│       └── user
│           ├── inmem.go
│           ├── interface.go
│           ├── service.go
│           └── service_test.go
├── app2
│   └── api
│       └── main.go
├── go.mod
└── internal
    ├── cache
    │   ├── interface.go
    │   ├── memcache.go
    │   └── mock.go
    ├── clock
    │   ├── clock.go
    │   ├── fake.go
    │   └── interface.go
    ├── compress
    │   └── zip.go
    ├── errors
    │   ├── errors.go
    │   └── errors_test.go
    ├── faker
    │   ├── animals.go
    │   ├── animals_test.go
    │   ├── interface.go
    │   └── mock.go
    ├── metric
    │   ├── interface.go
    │   └── prometheus.go
    ├── middleware
    │   ├── Cors.go
    │   ├── hasJWTAuthentication.go
    │   ├── hasJWTAuthentication_test.go
    │   ├── isJWTAuthenticated.go
    │   ├── isJWTAuthenticated_test.go
    │   ├── metrics.go
    │   ├── securityHeaders.go
    │   ├── validate.go
    │   ├── validateMultipart.go
    │   ├── validateMultipart_test.go
    │   └── validate_test.go
    ├── password
    │   ├── fake.go
    │   ├── interface.go
    │   └── password.go
    ├── pubsub
    │   ├── inmem.go
    │   ├── mock.go
    │   ├── pubsub.go
    │   └── sns.go
    ├── queue
    │   ├── inmem.go
    │   ├── mock.go
    │   ├── queue.go
    │   └── sqs.go
    ├── security
    │   ├── jwt.go
    │   └── jwt_test.go
    ├── storage
    │   ├── fs.go
    │   ├── inmem.go
    │   ├── interface.go
    │   └── s3.go
    └── test
        ├── cache_helper.go
        ├── postgresql_helper.go
```

## Erros

### Quem Consome Nossos Erros

A parte complicada sobre os erros é que eles precisam ser coisas diferentes para consumidores diferentes deles. Em qualquer sistema, temos pelo menos 3 papéis que são consumidores - a aplicação, o usuário final e a operação.

**O papel da aplicação**

Sua primeira linha de defesa no tratamento de erros é o próprio aplicativo. O código do seu aplicativo pode se recuperar de estados de erro rapidamente e sem chamar ninguém no meio da noite. No entanto, o tratamento de erros do aplicativo é o menos flexível e só pode tratar estados de erro bem definidos.

Um exemplo disso é o seu navegador recebendo um código de redirecionamento 301 e navegando para um novo local. É um processo contínuo que a maioria dos usuários ignora. É capaz de fazer isso porque a especificação HTTP têm códigos de erro bem definidos.

**Papel do usuário final**

Se a aplicação não for capaz de lidar com a condição de erro, esperamos que o usuário final possa resolver o problema. Seu usuário final pode ver um estado de erro como "Seu cartão de débito foi recusado" e é flexível o suficiente para resolver o problema (ou seja, depositar dinheiro em sua conta bancária).

Ao contrário da função do aplicativo, o usuário final precisa de uma mensagem legível que possa fornecer contexto para ajudá-lo a resolver o erro.

Esses usuários ainda estão limitados a erros bem definidos, pois revelar erros indefinidos pode comprometer a segurança do seu sistema. Por exemplo, um erro postgres pode detalhar informações de consulta ou esquema que podem ser usadas por um invasor. Quando confrontado com um erro indefinido, pode ser apropriado simplesmente dizer ao usuário para entrar em contato com o suporte técnico.

**Papel da operação**

Finalmente, a última linha de defesa é o operador do sistema, que pode ser um desenvolvedor ou uma pessoa de operações. Essas pessoas entendem os detalhes do sistema e podem trabalhar com qualquer tipo de erro.

Nesta função, você normalmente deseja ver o máximo de informações possível. Além do código de erro e da mensagem legível por humanos, um rastreamento de pilha lógico pode ajudar o operador a entender o fluxo do programa.

[Referência](https://middlemost.com/failure-is-your-domain/)

### Nossa estrutura de erros padrão

Dado o entendimento de que precisamos de códigos de erro, mensagens por humanos e rastreamento de pilha lógico, construímos um tipo de erro para lidar com os casos das nossas aplicações:

```go
package errors

// Application error codes.
const (
 ECONFLICT  = "conflict"  // action cannot be performed
 EINTERNAL  = "internal"  // internal error
 EINVALID   = "invalid"   // validation failed
 ENOTFOUND  = "not_found" // entity does not exist
 EFORBIDDEN = "forbidden" //operation forbidden
 EEXPECTED  = "expected"  //expected error that don't need to be logged
 ETIMEOUT   = "timeout"
)

// Error defines a standard application error.
type Error struct {
 Code    string // Machine-readable error code (papel da aplicação)
 Message string // Human-readable message (papel do usuário final)
 Op      string // Logical operation (papel da operação)
 Err     error  // Embedded error  (papel da operação)
 Detail  []byte // JSON encoded data  (papel da operação)
}
```

### Gerenciamento de erro pelo papel

**Papel da aplicação/operação**

Como estamos usando a Clean Architecture, as camadas de UseCase e Framework&Driver (Repositories, Queue,  etc) devem seguir as seguintes regras:

- Devem sempre retornar um erro caso exista, e não fazer log

- Devem sempre definir os valores para *Code*, *Op* e *Err*, de acordo com as definições a seguir

- Não há a necessidade de definir valor para o *Message* pois essa informação não vai chegar até o usuário final

**Regras para definir o campo Op**

É uma string com um dos formatos:

- *package.function*. Exemplo: *test.CreateServices*
- *package.receiver.function*. Exemplo: *address.MongoRepository.Find*

**Regras para definir o campo Code**

Escolher uma das opções a seguir, definidas no arquivo *internal/errors/errors.go*:

- ECONFLICT // action cannot be performed - exemplo: email duplicado
- EINTERNAL // internal error - Erros internos, referentes a própria linguagem ou ao servidor onde roda o código. Exemplo: salvar um arquivo, fazer o marshal de um json
- EINVALID // validation failed - Erros de lógica criadas por nós. Exemplo: salvar um usuário no banco de dados
- ENOTFOUND // entity does not exist
- EFORBIDDEN //operation forbidden
- EEXPECTED //expected error that don't need to be logged.

**Exemplos de erros:**

```go
//Find address na camada de repositório
func (r *MongoRepository) Find(id entity.ID) (*entity.Address, error) {
	result := entity.Address{}
	session := r.pool.Session(nil)
	defer session.Close()
	coll := session.DB(r.db).C("address")
	err := coll.Find(bson.M{"_id": id}).One(&result)

	if err != nil {
		return nil, &errors.Error{Op: "address.MongoRepository.Find", Err: err, Code: errors.ENOTFOUND}
	}
	return &result, nil
}
```

```go
//Find an address na camada de serviço
func (s *Service) Find(id entity.ID) (*entity.Address, error) {
	a, err := s.repo.Find(id) //está usando o MongoRepository
	if err != nil {
		return nil, &errors.Error{Op: "address.Service.Find", Err: err, Code: errors.ErrorCode(err)}
	}
	return a, nil
}
```

**Papel do usuário final**

De acordo com a Clean Architecture, a camada responsável pela interação com agentes externos (a UI neste caso) é a Controller. Desta forma, o campo Message deve ser definido nesta camada. Por exemplo, no arquivo ```app1/api/handler/address.go``` teríamos o código:

```go
a, err := services.Address.Find(entity.StringToID(id))
if err != nil {
 err.Message = "Erro lendo endereço"
 errorService.Log(err, elog.ERROR)
 errorService.RespondWithError(w, http.StatusNotFound, errors.ErrorCode(err), errors.ErrorMessage(err))
 return
}
```

A função ```errorService.Log``` (cujo código está dentro do pacote ```internal\elog```) faz o log do erro, enviando para o Sentry ou outro destino configurado no ambiente.

A função ```errorService.RespondWithError``` gera uma resposta para o cliente, com a respectiva mensagem de erro. O parâmetro ```w``` pode ser um ```http.ResponseWriter``` ou um ```io.Writer```, como o ```os.StdOut```.

### Log de erros

Para realizar o log dos erros o pacote ```elog``` usa o [```logrus```](https://github.com/sirupsen/logrus), que vai tratar o envio do log para destinos apropriados, de acordo com as configurações do ambiente (Sentry para staging e prod, stdout para ambiente de desenvolvimento).

A função ```Log``` recebe um ```error``` e um nível de severidade, que deve ser usado para a triagem das mensagens no storage de log. Os valores possíveis são:

**DEBUG**

Informações úteis para o desenvolvimento do software, que não são salvas, apenas visualizadas em tempo de desenvolvimento.

**INFO**

Informações relevantes e que devem ser salvas no registro de logs. É considerado um registro de nível de severidade baixo (LOW)

**WARNING**

Algo que aconteceu e deve ser revisado, mas que não impede o funcionamento do sistema. É considerado um registro de nível de severidade moderado (MODERATE). Exemplo: ao realizar o cadastro do usuário, o e-mail de boas vindas não pode ser enviado. Este registro deve ser enviado para o log como um WARNING

**ERROR**

Um erro aconteceu no sistema e deve ser revisado o mais rápido possível. É considerado um registro de nível de severidade alto (MAJOR). Exemplo: não foi possível realizar o cadastro do usuário

**FATAL**

O sistema parou de funcionar por algum motivo não esperado e deve ser revisado imediatamente. É considerado um registro de nível de severidade urgente (CRITICAL)

**PANIC**

O sistema não consegue iniciar por algum motivo não esperado e deve ser revisado imediatamente. É considerado um registro de nível de severidade urgente (CRITICAL)

[Refererência](https://www.softwaretestinghelp.com/how-to-set-defect-priority-and-severity-with-defect-triage-process/)

## Pacotes externos

### Router HTTP

Usamos o [Chi](https://github.com/go-chi/chi). Fatores usados na escolha:

- features (middlewares, inline middlewares, route groups and sub-router mounting; Context control);
- atividade do projeto (estrelas e atividade de atualizações do projeto no Github);
- usado por cases grandes, de acordo com o [site](https://github.com/go-chi/chi/issues/91) e pesquisas que realizamos;
- [performance](https://github.com/go-chi/chi#benchmarks)
- não possui dependências externas, *"plain ol' Go stdlib + net/http"*.

### Testes

Usamos o [testify](https://github.com/stretchr/testify) para deixar os códigos dos testes mais legíveis.

Exemplo de teste usando o testify:

```go
package yours

import (
 "testing"

 "github.com/stretchr/testify/assert"
)

func TestSomething(t *testing.T) {

 // assert equality
 assert.Equal(t, 123, 123, "they should be equal")

 // assert inequality
 assert.NotEqual(t, 123, 456, "they should not be equal")

 // assert for nil (good for errors)
 assert.Nil(t, object)

 // assert for not nil (good when you expect something)
 if assert.NotNil(t, object) {

  // now we know that object isn't nil, we are safe to make
  // further assertions without causing any errors
  assert.Equal(t, "Something", object.Value)

 }
}
```

### Mocks

Usamos o [testify/mock](https://github.com/stretchr/testify) e o [mockery](https://github.com/vektra/mockery) como solução para mocks em nossos testes.

Usamos essa [referência](https://blog.codecentric.de/2019/07/gomock-vs-testify/) para tomar a decisão. Este link também serve como introdução as principais features da solução.

### ORM X SQL

A ser definido.

## IDEs

Recomendamos o uso do Visual Studio Code, por ser usado também por todas as equipes da Trybe.

É necessário a instalação da [extensão oficial da linguagem Go](https://code.visualstudio.com/docs/languages/go).

### Sugestão de configuração do VS Code

```json
{
  "go.testFlags": [
    "-failfast",
    "-v"
  ],
  "go.toolsManagement.autoUpdate": true,
  "gopls": {
    "ui.semanticTokens": true
  },
  "go.lintOnSave": "file",
  "go.lintTool": "golint", 
  "go.formatTool": "goimports",
  "go.useLanguageServer": true,
  "[go]": {
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": true
    }
  },
  "go.docsTool": "godoc"
}
```
