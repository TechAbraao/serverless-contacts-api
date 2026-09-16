# API Serverless (Contacts API)

API REST serverless para consulta e gerenciamento inicial de contatos. A aplicação é executada em uma única AWS Lambda, com roteamento interno feito pelo `APIGatewayRestResolver`, da AWS Lambda Powertools.

O projeto pode ser testado localmente usando eventos no formato REST API v1 do API Gateway. Assim, é possível validar o comportamento da Lambda sem fazer deploy na AWS.

## Sumário

- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Instalação](#instalação)
- [Execução dos testes](#execução-dos-testes)
- [API](#api)
- [Eventos de teste](#eventos-de-teste)
- [AWS e Terraform](#aws-e-terraform)
- [Estrutura do projeto](#estrutura-do-projeto)
- [Estado atual](#estado-atual)

## Arquitetura

```text
API Gateway (REST API v1)
					|
					v
	 AWS Lambda: lambda_handler
					|
					v
APIGatewayRestResolver
					|
					v
	Rotas e middlewares
```

O fluxo principal está dividido em:

- `src/lambda_function.py`: ponto de entrada da Lambda e configuração do logger;
- `src/routes/contacts_routes.py`: definição das rotas de contatos;
- `src/middlewares/authorizations.py`: validação do header `Authorization`;
- `src/utils/`: respostas, erros, tipos de conteúdo e dados mockados;
- `events/`: eventos prontos para os cenários locais;
- `test/`: testes automatizados com `pytest`;
- `infra/terraform/`: configuração inicial da infraestrutura Terraform.

## Pré-requisitos

- Python `>= 3.14.4`;
- `pip` e `venv`;
- AWS CLI, apenas para trabalhar com recursos da AWS;
- Terraform `>= 0.14.4`, para a infraestrutura;
- credenciais AWS com as permissões necessárias, quando houver acesso à AWS.

O projeto usa, entre outras dependências, `aws-lambda-powertools` e `pytest`. As versões estão fixadas em [requirements.txt](requirements.txt).

## Instalação

Na raiz de `serverless-contacts-api`, crie um ambiente virtual e instale as dependências:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Para sair do ambiente virtual:

```bash
deactivate
```

## Execução dos testes

Execute todos os testes a partir da raiz do projeto:

```bash
pytest -q
```

Para executar somente os cenários de eventos do API Gateway:

```bash
pytest -q test/test_api_gateway_events.py
```

Também é possível executar o exemplo local que carrega [API_GATEWAY_PAYLOAD.json](API_GATEWAY_PAYLOAD.json) e imprime a resposta:

```bash
python -m test.test_local_lambda
```

O payload padrão representa uma chamada `GET /api/contacts` com autenticação válida.

## API

Todas as rotas abaixo exigem o header:

```http
Authorization: Bearer TokenJWTFake
```

O token acima é apenas um mock para desenvolvimento local. Ele não deve ser usado como mecanismo de autenticação em produção.

### `GET /api/contacts`

Retorna os cinco contatos mockados atualmente disponíveis.

Exemplo de evento:

```json
{
	"httpMethod": "GET",
	"path": "/api/contacts",
	"headers": {
		"Content-Type": "application/json",
		"Authorization": "Bearer TokenJWTFake"
	},
	"body": null,
	"isBase64Encoded": false
}
```

Resposta de sucesso: `200 OK`, com um corpo JSON contendo a propriedade `data`.

### `POST /api/contacts`

A rota está registrada e protegida por autenticação, mas ainda funciona como um esqueleto inicial: não valida nem persiste o corpo da requisição. A implementação de criação de contatos ainda precisa ser concluída antes de ser usada como endpoint de produção.

### Erros de autenticação

- Header ausente ou sem o prefixo `Bearer `: `401 Unauthorized`;
- Token diferente de `TokenJWTFake`: `401 Unauthorized`.

Os cenários estão cobertos por [INVALID_AUTH_HEADER.json](events/INVALID_AUTH_HEADER.json) e [INVALID_TOKEN.json](events/INVALID_TOKEN.json).

## Eventos de teste

O diretório [events](events) contém eventos do API Gateway prontos para uso:

| Arquivo | Cenário |
| --- | --- |
| [GET_API_CONTACTS_SUCCESS.json](events/GET_API_CONTACTS_SUCCESS.json) | Consulta de contatos com sucesso |
| [GET_CONTACT_BY_ID_SUCCESS.json](events/GET_CONTACT_BY_ID_SUCCESS.json) | Evento de consulta por identificador |
| [INVALID_AUTH_HEADER.json](events/INVALID_AUTH_HEADER.json) | Header de autorização ausente ou inválido |
| [INVALID_TOKEN.json](events/INVALID_TOKEN.json) | Token inválido |

Para adicionar um novo cenário, crie um JSON em `events/` e carregue-o em um teste usando a função `load_event` de [test/test_api_gateway_events.py](test/test_api_gateway_events.py).

## AWS e Terraform

O AWS CLI é necessário somente para operações que dependem da AWS, como consultar recursos ou realizar deploy. Instale-o conforme a [documentação oficial da AWS](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/cliv2-migration.html) e valide a instalação:

```bash
aws --version
```

Configure as credenciais com:

```bash
aws configure
```

Para o desenvolvimento local, não é necessário configurar credenciais AWS: os testes usam mocks e eventos locais.

A configuração Terraform disponível fica em `infra/terraform/inventories/dev`. Para inicializar e validar essa configuração:

```bash
cd infra/terraform/inventories/dev
terraform init
terraform validate
```

Defina a região por variável, por exemplo:

```bash
terraform plan -var="aws_region=us-east-1"
```

Revise o plano antes de aplicar qualquer alteração. Nunca versione credenciais, tokens ou arquivos `.tfvars` com informações sensíveis.

## Estrutura do projeto

```text
serverless-contacts-api/
├── API_GATEWAY_PAYLOAD.json
├── events/
├── infra/terraform/inventories/dev/
├── requirements.txt
├── src/
│   ├── lambda_function.py
│   ├── middlewares/
│   ├── routes/
│   ├── repositories/
│   └── utils/
└── test/
```

## Estado atual

- A consulta `GET /api/contacts` retorna dados mockados em memória;
- A autenticação é uma comparação local com um token fixo de teste;
- A rota `POST /api/contacts` ainda não persiste contatos;
- O repositório de contatos está reservado para a futura integração com uma camada de persistência;
- O Terraform contém atualmente a configuração inicial do provider AWS e da variável de região.