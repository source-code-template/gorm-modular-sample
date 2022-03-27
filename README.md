# gorm-modular-sample

#### To run the application
```shell
go run main.go
```

## Architecture
![Architecture](https://camo.githubusercontent.com/51415f43a296dfd2fce1beabef6cbc58a57c942ad75e2c286aed076f11757974/68747470733a2f2f63646e2d696d616765732d312e6d656469756d2e636f6d2f6d61782f3830302f312a4d306c4f32464a2d3877565067592d594d57396c75512e706e67)

#### Architecture with standard features: config, health check, logging, middleware log tracing
![Architecture](https://camo.githubusercontent.com/380ff8117dfbd6d032210c2c5bd797454d84ccd768a075015a9392d858c85041/68747470733a2f2f63646e2d696d616765732d312e6d656469756d2e636f6d2f6d61782f3830302f312a6a5156653852537a78525a4951417a346a71596270512e706e67)

#### [core-go/search](https://github.com/core-go/search)
- Build the search model at http handler
- Build dynamic SQL for search
  - Build SQL for paging by page index (page) and page size (limit)
  - Build SQL to count total of records

#### validator at [core-go/service](https://github.com/core-go/service)
- Check required, email, url, min, max, enum, regular expression
- Check phone number with country code
### Search users: Support both GET and POST 
#### POST /users/search
##### *Request:* POST /users/search
In the below sample, search users with these criteria:
- get users of page "1", with page size "20"
- email="tony": get users with email starting with "tony"
- dateOfBirth between "min" and "max" (between 1953-11-16 and 1976-11-16)
- sort by phone ascending, id descending
```json
{
    "page": 1,
    "limit": 20,
    "sort": "phone,-id",
    "email": "tony",
    "dateOfBirth": {
        "min": "1953-11-16T00:00:00+07:00",
        "max": "1976-11-16T00:00:00+07:00"
    }
}
```
##### GET /users/search?page=1&limit=2&email=tony&dateOfBirth.min=1953-11-16T00:00:00+07:00&dateOfBirth.max=1976-11-16T00:00:00+07:00&sort=phone,-id
In this sample, search users with these criteria:
- get users of page "1", with page size "20"
- email="tony": get users with email starting with "tony"
- dateOfBirth between "min" and "max" (between 1953-11-16 and 1976-11-16)
- sort by phone ascending, id descending

#### *Response:*
- total: total of users, which is used to calculate numbers of pages at client 
- list: list of users
```json
{
    "list": [
        {
            "id": "ironman",
            "username": "tony.stark",
            "email": "tony.stark@gmail.com",
            "phone": "0987654321",
            "dateOfBirth": "1963-03-24T17:00:00Z"
        }
    ],
    "total": 1
}
```

## API Design
### Common HTTP methods
- GET: retrieve a representation of the resource
- POST: create a new resource
- PUT: update the resource
- PATCH: perform a partial update of a resource
- DELETE: delete a resource

## API design for health check
To check if the service is available.
#### *Request:* GET /health
#### *Response:*
```json
{
    "status": "UP",
    "details": {
        "sql": {
            "status": "UP"
        }
    }
}
```

## API design for users
#### *Resource:* users

### Get all users
#### *Request:* GET /users
#### *Response:*
```json
[
    {
        "id": "spiderman",
        "username": "peter.parker",
        "email": "peter.parker@gmail.com",
        "phone": "0987654321",
        "dateOfBirth": "1962-08-25T16:59:59.999Z"
    },
    {
        "id": "wolverine",
        "username": "james.howlett",
        "email": "james.howlett@gmail.com",
        "phone": "0987654321",
        "dateOfBirth": "1974-11-16T16:59:59.999Z"
    }
]
```

### Get one user by id
#### *Request:* GET /users/:id
```shell
GET /users/wolverine
```
#### *Response:*
```json
{
    "id": "wolverine",
    "username": "james.howlett",
    "email": "james.howlett@gmail.com",
    "phone": "0987654321",
    "dateOfBirth": "1974-11-16T16:59:59.999Z"
}
```

### Create a new user
#### *Request:* POST /users 
```json
{
    "id": "wolverine",
    "username": "james.howlett",
    "email": "james.howlett@gmail.com",
    "phone": "0987654321",
    "dateOfBirth": "1974-11-16T16:59:59.999Z"
}
```
#### *Response:*
- status: can be configurable; 1: success, 0: duplicate key, 4: error
```json
{
    "status": 1,
    "value": {
        "id": "wolverine",
        "username": "james.howlett",
        "email": "james.howlett@gmail.com",
        "phone": "0987654321",
        "dateOfBirth": "1974-11-16T00:00:00+07:00"
    }
}
```
#### *Fail case sample:* 
- Request:
```json
{
    "id": "wolverine",
    "username": "james.howlett",
    "email": "james.howlett",
    "phone": "0987654321a",
    "dateOfBirth": "1974-11-16T16:59:59.999Z"
}
```
- Response: in this below sample, email and phone are not valid
```json
{
    "status": 4,
    "errors": [
        {
            "field": "email",
            "code": "email"
        },
        {
            "field": "phone",
            "code": "phone"
        }
    ]
}
```

### Update one user by id
#### *Request:* PUT /users/:id
```shell
PUT /users/wolverine
```
```json
{
    "username": "james.howlett",
    "email": "james.howlett@gmail.com",
    "phone": "0987654321",
    "dateOfBirth": "1974-11-16T16:59:59.999Z"
}
```
#### *Response:*
- status: can be configurable; 1: success, 0: duplicate key, 2: version error, 4: error
```json
{
    "status": 1,
    "value": {
        "id": "wolverine",
        "username": "james.howlett",
        "email": "james.howlett@gmail.com",
        "phone": "0987654321",
        "dateOfBirth": "1974-11-16T00:00:00+07:00"
    }
}
```

#### Problems for patch
If we pass a struct as a parameter, we cannot control what fields we need to update. So, we must pass a map as a parameter.
```go
type UserService interface {
    Update(ctx context.Context, user *User) (int64, error)
    Patch(ctx context.Context, user map[string]interface{}) (int64, error)
}
```
We must solve 2 problems:
1. At http handler layer, we must convert the user struct to map, with json format, and make sure the nested data types are passed correctly.
2. At service layer or repository layer, from json format, we must convert the json format to database format (in this case, we must convert to bson of Mongo)

#### Solutions for patch  
1. At http handler layer, we use [core-go/service](https://github.com/core-go/service), to convert the user struct to map, to make sure we just update the fields we need to update
```go
import server "github.com/core-go/service"

func (h *UserHandler) Patch(w http.ResponseWriter, r *http.Request) {
    var user User
    userType := reflect.TypeOf(user)
    _, jsonMap := sv.BuildMapField(userType)
    body, _ := sv.BuildMapAndStruct(r, &user)
    json, er1 := sv.BodyToJson(r, user, body, ids, jsonMap, nil)

    result, er2 := h.service.Patch(r.Context(), json)
    if er2 != nil {
        http.Error(w, er2.Error(), http.StatusInternalServerError)
        return
    }
    respond(w, result)
}
```

### Delete a new user by id
#### *Request:* DELETE /users/:id
```shell
DELETE /users/wolverine
```
#### *Response:* 1: success, 0: not found, -1: error
```json
1
```

## Common libraries
- [core-go/health](https://github.com/core-go/health): include HealthHandler, HealthChecker, SqlHealthChecker
- [core-go/config](https://github.com/core-go/config): to load the config file, and merge with other environments (SIT, UAT, ENV)
- [core-go/log](https://github.com/core-go/log): log and log middleware

### core-go/health
To check if the service is available, refer to [core-go/health](https://github.com/core-go/health)
#### *Request:* GET /health
#### *Response:*
```json
{
    "status": "UP",
    "details": {
        "sql": {
            "status": "UP"
        }
    }
}
```
To create health checker, and health handler
```go
    db, err := sql.Open(conf.Driver, conf.DataSourceName)
    if err != nil {
        return nil, err
    }

    sqlChecker := s.NewSqlHealthChecker(db)
    healthHandler := health.NewHealthHandler(sqlChecker)
```

To handler routing
```go
    r := mux.NewRouter()
    r.HandleFunc("/health", healthHandler.Check).Methods("GET")
```

### core-go/config
To load the config from "config.yml", in "configs" folder
```go
package main

import "github.com/core-go/config"

type Root struct {
    DB DatabaseConfig `mapstructure:"db"`
}

type DatabaseConfig struct {
    Driver         string `mapstructure:"driver"`
    DataSourceName string `mapstructure:"data_source_name"`
}

func main() {
    var conf Root
    err := config.Load(&conf, "configs/config")
    if err != nil {
        panic(err)
    }
}
```

### core-go/log *&* core-go/middleware
```go
import (
    "github.com/core-go/config"
    "github.com/core-go/log"
    m "github.com/core-go/middleware"
    "github.com/gorilla/mux"
)

func main() {
    var conf app.Root
    config.Load(&conf, "configs/config")

    r := mux.NewRouter()

    log.Initialize(conf.Log)
    r.Use(m.BuildContext)
    logger := m.NewLogger()
    r.Use(m.Logger(conf.MiddleWare, log.InfoFields, logger))
    r.Use(m.Recover(log.ErrorMsg))
}
```
To configure to ignore the health check, use "skips":
```yaml
middleware:
  skips: /health
```
