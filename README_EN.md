# The Trybe Go Style Guide.

Other language versions of the playbook:
[Portuguese (pt-BR)](./README.md)

This document defines the standards and best practices we adopt when writing Go code at Trybe. This guide is separated by theme, and prioritized by need and impact on the quality of our code.

> **Note:** This is a living document and reflects Trybe's needs, which may change over time, as well as the decisions we make to better solve our problems.

## How this guide is organized

- [Architecture](#architecture)
- [Configuration variables](#configuration-variables)
- [Repository structure](#Repository-structure)
- [Errors](#errors)
- [External packages](#external-packages)
- [IDEs](#ides)

## Architecture

### Clean Architecture

We organize our code using the Clean Architecture. More about the architecture and its concepts can be seen at the links:

[Clean Architecture, 2 years later](https://eltonminetto.dev/en/post/2020-07-06-clean-architecture-2years-later/)

[Software Architecture and the Clean Architecture](https://speakerdeck.com/eminetto/arquitetura-de-software-e-a-clean-architecture) - slides in pt-br

[[Arquitetura de software e a Clean Architecture - Hacktoberfest Brazil 2020](https://www.youtube.com/watch?v=pUl1eTTCcc4) - video in pt-br

## Configuration variables

According to the [recommendations](https://12factor.net/config) documented in the [12 factor](https://12factor.net) project we store the settings that change according to the environment (`staging`, `dev`, `production`) in environment variables. And settings that do not depend on the environment are stored in files in [TOML](https://toml.io/en/) format inside the repository. For easier configuration management we use the [Viper](https://github.com/spf13/viper) library.

## Repository structure

We use the monorepo concept, with the projects sharing the same repository on Github.

**Pros:**

- easy to manage dependencies and reuse code. As projects evolve we will create packages that are common to multiple projects (metrics, log, errors, etc) and having everything in the same repository makes reuse easier

**Cons:**

- More complex CI/CD since we would have to have the option to generate the binary and deploy different apps

We analyzed the pros and cons and came to the conclusion that monorepo is the best option for our projects.

Below is an example of how the repository is organized, with multiple projects.

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

## Errors

### Who Consumes Our Errors

The tricky part about errors is that they need to be different things for different consumers of them. In any system, we have at least 3 roles that are consumers - the application, the end user, and the operation.

**Application role**

Your first line of defense in error handling is your application itself. Your application code can recover from error states quickly and without paging anyone in the middle of the night. However, application error handling is the least flexible and it can only handle well-defined error states.

An example of this is your web browser receiving a 301 redirect code and navigating you to a new location. It’s a seamless process that most users are oblivious to. It’s able to do this because the HTTP specification has well-defined error codes.

**End user role**

If your application is unable to handle the error condition then hopefully your end user can resolve the issue. Your end user can see an error state such as "Your debit card is declined" and is flexible enough to resolve it (i.e. deposit money in their bank account).

Unlike the application role, the end user needs a human-readable message that can provide context to help them resolve the error.

These users are still limited to well-defined errors since revealing undefined errors could compromise the security of your system. For example, a postgres error may detail query or schema information that could be used by an attacker. When confronted with an undefined error, it may be appropriate to simply tell the user to contact technical support.

**Operator role**

Finally, the last line of defense is the system operator which may be a developer or an operations person. These people understand the details of the system and can work with any kind of error.

In this role, you typically want to see as much information as possible. In addition to the error code and human-readable message, a logical stack trace can help the operator understand the program flow.

[Reference](https://middlemost.com/failure-is-your-domain/)

### Error Pattern Structure

After understanding the reason for error code, messaging by humans, and logical stack tracing, we built an error type to handle the cases in our applications:

```go
package errors

import (
	"bytes"
	"encoding/json"
	"fmt"
)

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

### Error management by papers

**Application/Operation**

Using Clean Architecture, the UseCase and Framework&Driver layers (Repositories, Queue, etc) must follow these rules:

- Always return an error if one exists, and do not log

- Always set the values for *Code*, *Op* and *Err*, according to the following definitions

- There is no need to set a value for *Message* as this information will not reach the end user

**Define the Op field**

It is a string with one of the formats:

- *package.function*. Example: *test.CreateServices*
- *package.receiver.function*. Example: *address.MongoRepository.Find*

**Define the Code field**

Choose one of the following options defined in the file *internal/errors/errors.go*:

- ECONFLICT // action cannot be performed - example: duplicate email
- EINTERNAL // internal error - Internal errors, referring to the language itself or to the server where the code runs. Example: saving a file, marshaling a json
- EINVALID // validation failed - Logic errors created by us. Example: saving a user in the database
- ENOTFOUND // entity does not exist
- EFORBIDDEN //operation forbidden
- EEXPECTED //expected error that don't need to be logged.

**Examples:**

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

**End user**

Following the Clean Architecture definition, the layer responsible for interaction with external agents (the UI in this case) is the Controller. Thus, the Message field must be defined in this layer. For example, in the file ```app1/api/handler/address.go``` we would have the code:

```go
a, err := services.Address.Find(entity.StringToID(id))
if err != nil {
	err.Message = "Erro lendo endereço"
	errorService.Log(err, elog.ERROR)
	errorService.RespondWithError(w, http.StatusNotFound, errors.ErrorCode(err), errors.ErrorMessage(err))
	return
}
```

The ```errorService.Log``` function (whose code is inside the ```internalService.Log``` package) logs the error and sends it to Sentry or some other destination configured in the environment.

The ```errorService.RespondWithError``` function generates a response to the client, with a corresponding error message. The ```w``` parameter can be an ```http.ResponseWriter``` or an ```io.Writer```, such as ```os.StdOut```.

### Error Log

To log errors the ```elog``` package uses [```logrus```](https://github.com/sirupsen/logrus), which will handle sending the log to the appropriate destinations, according to the environment settings (Sentry for staging and prod, stdout for development environment).

The ```Log``` function is given a ```error``` and a severity level, which should be used for triaging messages in the log store. Possible values are:

**DEBUG**

Useful information for software development, which is not saved, but only visualized at development time.

**INFO**

Information that is relevant and should be saved in the log record. It is considered a low severity level log (LOW)

**WARNING**

Something that happened and should be reviewed, but does not prevent the system from working. It is considered a moderate severity level record (MODERATE). Example: when registering a user, the welcome email cannot be sent. This record must be sent to the log as a WARNING

**ERROR**

An error has occurred in the system and should be reviewed as soon as possible. It is considered a high severity level (MAJOR) record. Example: User registration not possible

**FATAL**

The system has stopped working for some unexpected reason and should be overhauled immediately. It is considered an urgent (CRITICAL) severity level record

**PANIC**

The system fails to start for some unexpected reason and should be reviewed immediately. It is considered an urgent (CRITICAL) severity level record

[Reference](https://www.softwaretestinghelp.com/how-to-set-defect-priority-and-severity-with-defect-triage-process/)

## External packages

### Router HTTP

We use [Chi](https://github.com/go-chi/chi). Factors used in the choice:

- features (middlewares, inline middlewares, route groups and sub-router mounting; Context control);
- project activity (stars and updates project activity on Github);
- used by large cases, according to the [website](https://github.com/go-chi/chi/issues/91) and research we have done;
- [performance](https://github.com/go-chi/chi#benchmarks)
- has no external dependencies, *"plain ol' Go stdlib + net/http"*.

### Tests

We use [testify](https://github.com/stretchr/testify) to make the test code more readable.

Example of a test using testify:

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

We use [testify/mock](https://github.com/stretchr/testify) and [mockery](https://github.com/vektra/mockery) as a solution for mocks in our tests.

We used this [reference](https://blog.codecentric.de/2019/07/gomock-vs-testify/) to make the decision. This link also serves as an introduction to the main features of the solution.

### ORM X SQL

To be defined.

## IDEs

We recommend using Visual Studio Code, as it is also used by all Trybe teams.

The installation of the [official Go language extension](https://code.visualstudio.com/docs/languages/go) is required.

### VS Code configuration suggestion

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
