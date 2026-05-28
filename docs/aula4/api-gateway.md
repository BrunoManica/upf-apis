# API Gateway

## Objetivo da aula

Vamos entender o que é um API Gateway e por que ele aparece quando uma aplicação deixa de ter apenas uma API e passa a ter vários serviços.

Nesta aula, o foco é simples: entender o papel do gateway antes de escrever o código.

A ideia mais importante é esta: o API Gateway fica na entrada do sistema e encaminha as requisições para os serviços corretos.

## Resultado final

Ao final desta aula, você deve conseguir explicar:

* por que o cliente chama o gateway em vez de chamar cada microsserviço diretamente;
* como o gateway decide para qual serviço enviar uma requisição;
* por que autenticação, rate limiting e cache costumam aparecer nessa camada;
* o que não deve ser colocado dentro do gateway.

## Contexto

Em uma aplicação pequena, normalmente temos uma API só.

Algo assim:

```text
Cliente -> API -> Banco de dados
```

O cliente chama a API, a API processa a requisição e devolve a resposta. Para começar, isso é simples e funciona bem.

O problema aparece quando o sistema cresce e começa a ser dividido em serviços menores:

```text
Cliente -> Serviço de Usuários
Cliente -> Serviço de Produtos
Cliente -> Serviço de Pagamentos
Cliente -> Serviço de Pedidos
```

Agora o cliente precisa saber o endereço de cada serviço.

Isso cria alguns problemas:

* O cliente precisa conhecer várias portas e vários endereços.
* Cada serviço pode ter uma regra de segurança diferente.
* Fica mais difícil mudar um serviço sem afetar quem consome a API.
* A aplicação cliente começa a saber detalhes internos demais do backend.

O API Gateway aparece para resolver esse tipo de problema.

Ele vira a porta de entrada da aplicação.

## Explicação conceitual

Um API Gateway é uma aplicação que fica entre o cliente e os serviços internos.

Ele recebe uma requisição, olha para informações como caminho, método HTTP, headers ou domínio, e decide para qual serviço essa requisição deve ir.

Um exemplo simples:

```text
GET /api/users    -> Serviço de Usuários
GET /api/products -> Serviço de Produtos
POST /api/orders  -> Serviço de Pedidos
```

O gateway não precisa conter a regra de negócio de usuários, produtos ou pedidos.

Essa parte é importante.

O gateway não deve virar um lugar onde toda a lógica da aplicação fica misturada. Ele existe principalmente para receber, filtrar, rotear e controlar as chamadas.

Os serviços continuam responsáveis pelas regras de negócio.

### Sem API Gateway

Sem gateway, o cliente precisa chamar cada serviço diretamente:

```text
GET http://localhost:8081/users
GET http://localhost:8082/products
GET http://localhost:8083/orders
```

Isso até funciona em um exemplo pequeno, mas começa a ficar ruim quando o número de serviços aumenta.

Imagine também que amanhã o serviço de usuários mude de porta ou vá para outro servidor. O cliente teria que mudar junto.

### Com API Gateway

Com gateway, o cliente chama apenas um endereço:

```text
GET http://localhost:8080/api/users
GET http://localhost:8080/api/products
GET http://localhost:8080/api/orders
```

O gateway recebe a chamada e encaminha internamente:

```text
/api/users    -> http://localhost:8081/users
/api/products -> http://localhost:8082/products
/api/orders   -> http://localhost:8083/orders
```

Para o cliente, existe uma entrada só.

Para o backend, continuam existindo serviços separados.

## Setup inicial

Antes de pensar em ferramenta, framework ou linguagem, pense nos componentes:

```text
gateway-demo/
├── Serviço de Usuários
├── Serviço de Produtos
└── API Gateway
```

Cada parte tem uma responsabilidade:

* Serviço de Usuários: responde dados e regras de usuários.
* Serviço de Produtos: responde dados e regras de produtos.
* API Gateway: recebe as chamadas externas e encaminha para os serviços.

Nesta aula conceitual, vamos começar pelo roteamento e depois olhar para três recursos que aparecem muito quando falamos de gateway: autenticação, rate limiting e cache.

Esses recursos têm relação direta com a ideia de porta de entrada.

Se toda chamada externa passa pelo gateway, faz sentido usar esse ponto para aplicar algumas regras antes de deixar a requisição chegar nos serviços internos.

## Passo a passo

### 1. Entender quem faz o quê

Pense em três aplicações separadas:

```text
API Gateway       -> porta de entrada
Serviço Usuários  -> regra de usuários
Serviço Produtos  -> regra de produtos
```

Essa separação é o primeiro passo para pensar em microsserviços.

O gateway não substitui os serviços.

Ele apenas organiza a entrada para chegar neles.

### 2. Entender o roteamento

Roteamento é a decisão de encaminhar uma requisição para o destino certo.

Uma rota pode ser lida assim:

```text
Quando chegar uma requisição em /api/users/**
encaminhe para o Serviço de Usuários
```

Outra rota:

```text
Quando chegar uma requisição em /api/products/**
encaminhe para o Serviço de Produtos
```

O gateway usa alguma condição para decidir a rota.

A condição mais comum é o caminho da URL:

```text
/api/users
/api/products
/api/orders
```

Mas em sistemas reais a decisão também pode considerar:

* Método HTTP, como `GET`, `POST`, `PUT` ou `DELETE`.
* Headers da requisição.
* Domínio usado pelo cliente.
* Versão da API.
* Tipo de cliente, como web, mobile ou parceiro externo.

Para começar, o caminho da URL já é suficiente.

### 3. Entender filtros

Além de rotear, o gateway pode aplicar filtros.

Filtro é uma etapa que acontece antes ou depois de encaminhar a requisição.

Um exemplo simples é remover um prefixo da URL.

O cliente chama:

```text
GET /api/users
```

Mas o serviço interno espera:

```text
GET /users
```

Então o gateway pode transformar o caminho:

```text
/api/users      -> /users
/api/users/123  -> /users/123
```

Esse tipo de transformação ajuda a manter uma URL organizada para o cliente sem obrigar todos os serviços internos a usarem exatamente o mesmo prefixo.

Outros filtros comuns são:

* Adicionar ou remover headers.
* Registrar logs.
* Validar autenticação.
* Bloquear chamadas acima de um limite.
* Redirecionar chamadas para uma nova versão da API.

Nem tudo precisa ser usado na primeira aula.

Primeiro, o mais importante é entender o caminho da requisição.

### 4. Entender autenticação no gateway

Autenticação é o processo de descobrir quem está fazendo a requisição.

Em APIs modernas, isso normalmente aparece com um token enviado no header:

```text
Authorization: Bearer token-aqui
```

O gateway pode verificar esse token antes de encaminhar a requisição para um serviço interno.

Na prática, isso evita repetir a mesma verificação básica em todos os serviços.

Um exemplo:

```text
GET /api/users
Authorization: Bearer token-valido

Gateway valida o token.
Gateway encaminha para users-service.
```

Se o token estiver ausente ou inválido:

```text
Gateway responde 401 Unauthorized.
users-service nem recebe a requisição.
```

Isso melhora a segurança porque o sistema barra chamadas inválidas logo na entrada.

Mesmo assim, isso não significa que os serviços internos podem ser descuidados.

Em produção, é comum combinar duas proteções:

* o gateway valida a chamada externa;
* os serviços internos continuam protegidos contra acesso direto indevido.

Use autenticação no gateway quando você precisa centralizar a entrada de usuários, aplicativos mobile, frontends web ou sistemas parceiros.

### 5. Entender rate limiting

Rate limiting significa limitar a quantidade de requisições que um cliente pode fazer em um intervalo de tempo.

Por exemplo:

```text
Cliente comum: 60 requisições por minuto
Cliente parceiro: 600 requisições por minuto
```

Isso existe porque uma API não aguenta tráfego infinito.

Sem limite, um cliente com bug, um script mal configurado ou uma tentativa de ataque pode mandar milhares de chamadas e prejudicar todo o sistema.

Com rate limiting, o gateway consegue responder antes de sobrecarregar os serviços:

```text
Requisições dentro do limite -> seguem para o serviço
Requisições acima do limite  -> recebem 429 Too Many Requests
```

O status `429 Too Many Requests` quer dizer: a requisição até pode estar correta, mas passou do limite permitido naquele momento.

Use rate limiting no gateway quando você precisa proteger os serviços internos, controlar consumo por cliente ou evitar abuso em endpoints públicos.

### 6. Entender cache no gateway

Cache é guardar uma resposta por um tempo para evitar buscar a mesma informação toda hora no serviço interno.

Imagine uma rota de produtos:

```text
GET /api/products
```

Se várias pessoas chamam essa rota em poucos segundos e os produtos quase não mudam, o gateway pode guardar a resposta por um curto período.

O primeiro cliente faz a chamada:

```text
Cliente -> Gateway -> products-service
```

Depois, enquanto o cache ainda vale:

```text
Cliente -> Gateway -> resposta em cache
```

Isso reduz latência e diminui carga nos serviços internos.

Mas cache precisa de cuidado.

Não faz sentido cachear qualquer coisa.

Bons candidatos:

* listas públicas de produtos;
* categorias;
* configurações públicas;
* dados que mudam pouco.

Evite cachear sem pensar:

* dados de usuário autenticado;
* carrinho de compras;
* saldo, pagamento ou informação sensível;
* respostas que mudam a cada requisição.

Use cache no gateway quando a resposta é pública ou pouco variável e quando devolver uma versão levemente antiga por alguns segundos não prejudica o sistema.

### 7. Entender o que o gateway não deve fazer

O gateway não deve ser o lugar principal da regra de negócio.

Ele não deve decidir:

* Como cadastrar um usuário.
* Como calcular o preço de um produto.
* Como aprovar um pagamento.
* Como atualizar o estoque.

Essas regras pertencem aos serviços.

O gateway cuida da entrada.

Os serviços cuidam do negócio.

Essa divisão evita que o gateway vire uma aplicação enorme e difícil de manter.

## Código completo

O código completo da aula prática está no tutorial:

[Tutorial prático de API Gateway](tutorial-api-gateway.md)

## Erros comuns

### Tentar colocar regra de negócio no gateway

O gateway não deve decidir regra de cadastro, cálculo, estoque ou pagamento.

Essas regras pertencem aos serviços.

O gateway deve cuidar da entrada, do roteamento e de controles transversais, como autenticação, logs e limites de requisição.

### Confundir gateway com microsserviço de negócio

O gateway também é uma aplicação, mas a função dele é diferente.

Ele não é o serviço de usuários.

Ele não é o serviço de produtos.

Ele é a entrada para chegar nesses serviços.

### Começar com recursos demais

É comum tentar colocar autenticação, banco, cache, observabilidade e limite de requisição no primeiro exemplo.

Esses recursos são importantes, mas precisam entrar depois que o roteamento estiver claro.

Primeiro o aluno precisa enxergar a requisição saindo do cliente, passando pelo gateway e chegando no serviço certo.

Depois disso, os outros recursos fazem mais sentido.

### Usar cache em resposta sensível

Cache no gateway pode melhorar desempenho, mas também pode causar problema sério se for usado no lugar errado.

Não faça cache de dados privados de usuário, tokens, saldo, pagamentos ou qualquer resposta que dependa da identidade de quem chamou.

### Achar que rate limiting substitui segurança

Rate limiting limita volume de chamadas.

Ele não prova identidade, não autoriza acesso e não valida regra de negócio.

Para isso, você ainda precisa de autenticação, autorização e validações dentro dos serviços.

## Resumo

O API Gateway é a porta de entrada de uma arquitetura com vários serviços.

Ele ajuda o cliente a chamar apenas um endereço, enquanto o backend continua dividido em serviços menores.

Além de rotear, ele pode aplicar controles transversais:

* autenticação, para barrar chamadas sem identidade válida;
* rate limiting, para controlar excesso de requisições;
* cache, para acelerar respostas que podem ser reaproveitadas com segurança.

Nesta aula, o ponto mais importante é este:

```text
O cliente chama o gateway.
O gateway encaminha para o serviço certo.
O serviço continua com a regra de negócio.
```

Depois que esse conceito estiver claro, a aula prática pode escolher uma ferramenta específica para implementar esse comportamento.


