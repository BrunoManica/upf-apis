# Material preparatório — Docker e ambiente local

## Objetivo

Ao final deste material, você deverá conseguir explicar a relação entre imagem, container e Docker Compose e preparar, de forma reproduzível, um MongoDB local para a API de usuários da Aula 1.

## Resultado final

Você terá um arquivo `docker-compose.yml` que inicia um MongoDB em container. Ao executar os comandos de verificação, verá o container ativo e confirmará que o banco está disponível na porta `27017` da sua máquina.

Use também os [slides sobre containers e máquinas virtuais](../slides/containers-vs-vms-slides.html) para acompanhar a explicação conceitual em sala.

## Contexto

Na próxima aula construiremos uma API de usuários com Java 17 e Spring Boot. Essa API precisa de um MongoDB. Instalar e configurar o banco manualmente em cada computador costuma produzir diferenças de versão, serviço, senha e configuração. É assim que aparece o conhecido problema: “na minha máquina funciona”.

O Docker permite descrever o ambiente necessário em um arquivo e repeti-lo. Assim, cada estudante inicia a mesma versão do MongoDB com o mesmo comando. Neste material, o Docker prepara o ambiente; a API em Java 17 será criada na Aula 1.

## Explicação conceitual

Antes de executar comandos, leia também [Containers vs VMs](containers-vs-vms.md). A comparação explica por que um container é uma opção prática para o ambiente local.

### Imagem, container e Dockerfile

Podemos usar uma analogia de bolos:

| Conceito | Analogia | O que significa no Docker |
| --- | --- | --- |
| Dockerfile | Receita | Arquivo com as instruções para produzir uma imagem. |
| Imagem | Receita já preparada para ser repetida | Pacote versionado que contém o necessário para iniciar um programa. |
| Container | Bolo pronto em uso | Instância em execução criada a partir de uma imagem. |

Uma mesma imagem pode gerar vários containers. Nesta aula, usaremos a imagem oficial `mongo:7`. O Docker baixa essa imagem uma vez e cria o container que executa o banco.

O Docker não virtualiza uma máquina completa para cada container. No Linux, os containers compartilham o kernel do sistema hospedeiro e mantêm processos, rede e arquivos isolados. No Docker Desktop, usado normalmente no Windows, existe uma camada de virtualização para disponibilizar esse ambiente Linux. Por isso, “container” não significa que todos os sistemas operacionais sejam idênticos; significa que a configuração descrita pode ser reproduzida pelo Docker.

### Docker Compose

O Docker Compose descreve serviços relacionados em um arquivo YAML. Em vez de decorar uma sequência longa de comandos, você registra a imagem, a porta e os dados persistentes no `docker-compose.yml`. Depois, `docker compose up -d` cria ou inicia o ambiente.

Hoje haverá apenas um serviço, o MongoDB. Nas próximas aulas, o mesmo arquivo poderá incluir a API de usuários e outras dependências. O comando atual é `docker compose`, com espaço; ele substitui o formato antigo `docker-compose`.

## Preparação

Instale o Docker Desktop no Windows ou o Docker Engine com o plugin Docker Compose no Linux. Abra o Docker Desktop, quando utilizá-lo, antes de continuar.

Você não precisa instalar MongoDB diretamente no computador. Também não precisa criar um projeto Java neste material. A Aula 1 usará Java 17, Spring Boot e o banco iniciado aqui.

No terminal, confirme que o cliente Docker e o Compose estão disponíveis:

```bash
docker --version
docker compose version
```

Os dois comandos devem imprimir suas versões. Em seguida, confirme que o Docker consegue responder:

```bash
docker info
```

Esse comando mostra informações do cliente e do servidor Docker. Se ele terminar com informações do servidor, o ambiente está pronto para criar containers.

## Passo a passo

### 1. Criar a pasta do ambiente

Crie uma pasta para os arquivos da disciplina e entre nela. Escolha um local que você consiga encontrar depois, pois a Aula 1 poderá reutilizá-la.

```bash
mkdir ambiente-api-usuarios
cd ambiente-api-usuarios
```

No Windows, você pode executar esses comandos no PowerShell ou criar a pasta pelo Explorador de Arquivos e abri-la no terminal.

### 2. Criar o arquivo Docker Compose

Na pasta criada, crie um arquivo chamado `docker-compose.yml` com este conteúdo completo:

```yaml
services:
  mongo:
    image: mongo:7
    container_name: mongo-api-usuarios
    ports:
      - "27017:27017"
    volumes:
      - mongo_api_usuarios_data:/data/db

volumes:
  mongo_api_usuarios_data:
```

O serviço se chama `mongo`. Ele usa a imagem `mongo:7`, publica a porta padrão do banco e cria o volume nomeado `mongo_api_usuarios_data`.

O volume é importante: se o container for parado e iniciado outra vez, os dados do MongoDB continuam disponíveis. Remover um container não é o mesmo que remover o volume; os dados só seriam apagados se você pedisse essa remoção explicitamente.

### 3. Conferir a configuração antes de iniciar

Ainda na pasta que contém o arquivo, execute:

```bash
docker compose config
```

O Docker Compose imprime a configuração final. Confira se o serviço `mongo`, a porta `27017` e o volume aparecem na saída. Esse comando não inicia containers.

### 4. Iniciar o MongoDB

Execute:

```bash
docker compose up -d
```

Na primeira execução, o Docker baixa a imagem do MongoDB. Em seguida, cria e inicia o container em segundo plano; `-d` significa que o terminal fica livre para o próximo comando.

### 5. Verificar o container

Liste os containers em execução:

```bash
docker ps
```

Você deve encontrar uma linha com o nome `mongo-api-usuarios`, a imagem `mongo:7` e o mapeamento de porta semelhante a `0.0.0.0:27017->27017/tcp`.

Depois, consulte o estado definido pelo Compose:

```bash
docker compose ps
```

O serviço `mongo` deve aparecer em execução. A inicialização do banco pode levar alguns segundos. Para observar as mensagens já emitidas sem acompanhar o log continuamente, execute:

```bash
docker compose logs --tail=20 mongo
```

Se a mensagem de disponibilidade ainda não apareceu, aguarde alguns segundos e execute o comando novamente. Em seguida, confirme que o próprio MongoDB responde a uma consulta simples:

```bash
docker compose exec mongo mongosh --quiet --eval "db.runCommand({ ping: 1 })"
```

A saída deve conter `ok: 1`. Agora o ambiente local está preparado para a Aula 1.

### 6. Parar e retomar o ambiente

Quando encerrar o estudo, pare o container sem apagar seus dados:

```bash
docker compose stop
```

Para retomá-lo posteriormente, na mesma pasta, execute:

```bash
docker compose start
```

Se quiser encerrar e remover somente o container e a rede criados pelo Compose, mantendo o volume de dados, execute:

```bash
docker compose down
```

Não use comandos de limpeza geral, como `docker system prune -a`, como rotina de estudo. Eles podem remover imagens e outros recursos que você utiliza em projetos diferentes.

## Código completo

O único arquivo criado nesta aula é o `docker-compose.yml`:

```yaml
services:
  mongo:
    image: mongo:7
    container_name: mongo-api-usuarios
    ports:
      - "27017:27017"
    volumes:
      - mongo_api_usuarios_data:/data/db

volumes:
  mongo_api_usuarios_data:
```

Estrutura final da pasta:

```text
ambiente-api-usuarios/
└── docker-compose.yml
```

## Resumo

- Uma imagem é um pacote reutilizável; um container é sua execução concreta.
- Docker Compose registra o ambiente em arquivo e o inicia com `docker compose up -d`.
- O serviço `mongo` deixa o MongoDB acessível em `localhost:27017` para a futura API de usuários.
- O volume nomeado preserva os dados quando o container é parado ou recriado sem remoção de volumes.

Na Aula 1, você criará a API de usuários em Java 17 e configurará a aplicação para se conectar a esse MongoDB.

## Referências

- [Docker: visão geral](https://docs.docker.com/get-started/docker-overview/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Imagem oficial do MongoDB](https://hub.docker.com/_/mongo)
