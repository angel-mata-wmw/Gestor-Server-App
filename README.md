# Gestor de Clientes — Servidor

API REST para sincronização de clientes da aplicação mobile, com validação de CPF/CNPJ, controle de duplicidade e tratamento de erros centralizado.

## Tecnologias

- **Java 25**
- **Spring Boot 4.0.5**
  - Spring Web MVC
  - Spring Data JPA
  - Spring Validation
  - Spring DevTools
- **Banco de dados:** H2 (in-memory)
- **Build:** Maven

## Estrutura do projeto

```
src/
└── main/java/br/com/drky/gestor/
    ├── controller/     # Endpoints REST (ClienteController)
    ├── service/        # Regras de negócio e validação de CPF/CNPJ (ClienteService)
    ├── repository/     # Acesso ao banco via JPA (ClienteRepository)
    ├── model/          # Entidade Cliente e enum TipoCliente
    ├── dto/            # DTOs de request e response
    ├── exception/      # Exceções de domínio
    └── validation/     # Handler global de erros (InvalidObjectErrorHandler)
```

## Endpoints

Base URL: `http://localhost:8080`

| Método | Rota | Descrição |
|---|---|---|
| GET | `/clientes` | Lista todos os clientes |
| GET | `/clientes/findById/{id}` | Busca cliente por ID |
| POST | `/clientes` | Cadastra novo cliente |
| POST | `/clientes/verificar` | Verifica se um CPF/CNPJ já está cadastrado |
| PUT | `/clientes/{id}` | Atualiza telefone ou e-mail |
| DELETE | `/clientes/{id}` | Remove cliente por ID |

## Exemplos de uso

### Cadastrar cliente (pessoa física)

```http
POST /clientes
Content-Type: application/json

{
  "codigo": "1",
  "nome": "João Silva",
  "tipo": "FISICO",
  "cpfCnpj": "529.982.247-25",
  "telefone": "34999990000",
  "email": "joao@email.com",
  "sincronizado": "false"
}
```

### Cadastrar cliente (pessoa jurídica)

```http
POST /clientes
Content-Type: application/json

{
  "codigo": "2",
  "nome": "Empresa LTDA",
  "tipo": "JURIDICO",
  "cpfCnpj": "11.222.333/0001-81",
  "telefone": "34977770000",
  "email": "contato@empresa.com",
  "sincronizado": "false"
}
```

### Atualizar cliente

```http
PUT /clientes/1
Content-Type: application/json

{
  "telefone": "34988888888",
  "email": "novo@email.com"
}
```

### Verificar se CPF/CNPJ já está cadastrado

```http
POST /clientes/verificar
Content-Type: application/json

{
  "tipo": "FISICO",
  "documento": "529.982.247-25"
}
```

Retorna `true` se já cadastrado, `false` se disponível.

## Regras de negócio

- O campo `tipo` aceita `FISICO` ou `JURIDICO` (case-insensitive).
- O CPF/CNPJ é validado matematicamente antes do cadastro.
- Não é permitido cadastrar dois clientes com o mesmo CPF ou CNPJ.
- O campo `email` é opcional no cadastro.
- Na atualização, apenas `telefone` e `email` podem ser alterados.
- Ao receber um cliente do mobile, o campo `sincronizado` é definido como `true` no servidor.

## Respostas de erro

| Situação | Status HTTP | Mensagem |
|---|---|---|
| Campo obrigatório ausente | `400` | Lista de campos inválidos |
| Tipo de cliente inválido | `400` | `"Tipo de cliente invalido"` |
| CPF inválido | `400` | `"CPF inválido"` |
| CNPJ inválido | `400` | `"CNPJ inválido"` |
| CPF já cadastrado | `400` | `"CPF já cadastrado"` |
| CNPJ já cadastrado | `400` | `"CNPJ já cadastrado"` |
| Cliente não encontrado | `404` | `"Cliente não encontrado"` |
| Dados inválidos na requisição | `400` | `"Dados invalidos"` |

## DTOs

### `CreateRequestClienteDTO` — entrada para cadastro

| Campo | Obrigatório | Descrição |
|---|---|---|
| `codigo` | Sim | ID do cliente no mobile |
| `nome` | Sim | Nome ou razão social |
| `tipo` | Sim | `FISICO` ou `JURIDICO` |
| `cpfCnpj` | Sim | CPF ou CNPJ formatado |
| `telefone` | Sim | Número de telefone |
| `email` | Não | E-mail (pode ser nulo) |
| `sincronizado` | Não | Status de sincronização |

### `UpdateRequestClienteDTO` — entrada para atualização

| Campo | Obrigatório | Descrição |
|---|---|---|
| `telefone` | Não | Novo telefone |
| `email` | Não | Novo e-mail |

### `ResponseClienteDTO` — saída padrão

| Campo | Descrição |
|---|---|
| `codigo` | ID do cliente |
| `nome` | Nome ou razão social |
| `tipo` | `FISICO` ou `JURIDICO` |
| `cpfCnpj` | CPF ou CNPJ |
| `telefone` | Telefone |
| `email` | E-mail (pode ser nulo) |
| `sincronizado` | `"true"` ou `"false"` |
| `excluido` | `"true"` ou `"false"` |

## Testes

Os testes cobrem `ClienteController` e `ClienteService`.

### ClienteServiceTest — testes de regras de negócio

| Caso de teste | Resultado esperado |
|---|---|
| CPF válido | `true` |
| CPF inválido | `false` |
| CNPJ válido | `true` |
| CNPJ inválido | `false` |
| CPF já cadastrado | lança `RegisteredClientException` com `"CPF já cadastrado"` |
| CNPJ já cadastrado | lança `RegisteredClientException` com `"CNPJ já cadastrado"` |
| Atualizar cliente inexistente | lança `ClientNotFoundException` com `"Cliente não encontrado"` |
| Buscar cliente inexistente por ID | lança `ClientNotFoundException` com `"Cliente não encontrado"` |
| Buscar cliente existente por ID | retorna o cliente correto |
| Listar todos os clientes | retorna lista ordenada por código |
| Excluir cliente inexistente | lança `ClientNotFoundException` com `"Cliente não encontrado"` |

### ClienteControllerTest — testes de integração via MockMvc

| Ordem | Caso de teste | Status esperado |
|---|---|---|
| 1 | Listagem de clientes | `200` |
| 2 | Busca por ID existente | `200` + nome correto |
| 3 | Busca com ID inválido (letra) | `400` |
| 4 | Busca por ID inexistente | `404` |
| 5 | Cadastro bem-sucedido | `200` + `"Cliente criado com sucesso!!"` |
| 6 | Cadastro com corpo inválido | `400` |
| 7 | CPF duplicado no cadastro | `400` |
| 8 | CPF inválido no cadastro | `400` |
| 9 | CNPJ duplicado no cadastro | `400` |
| 10 | CNPJ inválido no cadastro | `400` |
| 11 | Atualização bem-sucedida | `200` + `"Cliente atualizado com sucesso!!"` |
| 12 | Atualização sem campos obrigatórios | `400` |
| 13 | Atualização com ID inexistente | `400` |
| 14 | Exclusão bem-sucedida | `200` + `"Cliente excluido com sucesso!!"` |
| 15 | Exclusão com ID inválido | `400` |

## Como executar

### Pré-requisitos

- Java 25+
- Maven 3.x

### Rodando a aplicação

```bash
# Clone o repositório
git clone https://github.com/DarkeyS24/Gestor-Project-Server.git
cd Gestor-Project-Server

# Execute com Maven Wrapper
./mvnw spring-boot:run
```

A aplicação sobe em `http://localhost:8080`.

### Rodando os testes

```bash
./mvnw test
```
