# Aula 2 — SOLID com Java 17 e Spring Boot

---

## Objetivo da aula

Ao final desta aula, você deverá conseguir:

- entender qual problema o SOLID tenta resolver;
- identificar responsabilidades dentro de uma aplicação;
- perceber quando uma classe começa a assumir responsabilidades demais;
- usar interfaces para reduzir acoplamento quando isso fizer sentido;
- aplicar SRP, OCP, LSP, ISP e DIP em situações práticas;
- utilizar Lombok para reduzir código repetitivo;
- utilizar `@Builder` para construção de objetos;
- entender que SOLID não significa criar abstrações e interfaces sem necessidade.

> SOLID existe para melhorar o design do software, não para aumentar o número de arquivos.

---

## Resultado final

Ao concluir a prática, você terá uma API simples de pedidos organizada em `controller`, `service` e `repository`. A partir de problemas concretos no código, você também criará políticas de desconto e formas de entrega que mostram onde cada princípio SOLID pode ajudar.

O resultado mais importante não é a quantidade de interfaces ou classes. É conseguir justificar cada separação com uma dificuldade real de manutenção.

---

## Contexto

Na Aula 1, você construiu uma API REST e conheceu a separação entre controller, service e repository. Agora, você observará o que acontece quando regras diferentes começam a se acumular nas mesmas classes.

Nesta aula, o domínio de pedidos será usado apenas como um cenário reduzido para praticar SOLID. O armazenamento ficará em memória para que a atenção permaneça nas responsabilidades, nos contratos e nas dependências do código.

---

## 1. Preparação do projeto

Vamos construir uma API extremamente simples de pedidos.

### Stack

- Java 17
- Spring Boot
- Spring Web
- Lombok
- armazenamento em memória
- sem banco de dados
- sem JPA
- sem autenticação
- sem arquitetura complexa

O objetivo da aula é SOLID. Todo elemento que não ajuda a ensinar SOLID deve ser mantido simples.

---

## 2. Domínio

Teremos apenas duas classes principais:

```text
Pedido
 ├── id
 ├── cliente
 ├── itens
 └── status

ItemPedido
 ├── produto
 ├── quantidade
 └── valorUnitario
```

Endpoints:

```http
POST /pedidos
GET /pedidos
GET /pedidos/{id}
```

---

## 3. Criando os modelos com records e Lombok

Nesta prática, os objetos que apenas transportam dados serão `record`. Um record já fornece construtor, métodos de acesso, `equals`, `hashCode` e `toString`. Como seus componentes são imutáveis, não existem setters.

Controllers, services, repositories e estratégias continuam sendo classes. Eles possuem comportamento, dependências injetadas ou estado interno e, portanto, não são simples transportadores de dados. Usar `record` em todos os arquivos apenas para eliminar a palavra `class` não seria uma aplicação adequada do recurso.

Todos os caminhos apresentados a seguir são relativos à raiz do projeto. Crie os diretórios quando eles ainda não existirem. Quando a aula disser **substitua o conteúdo**, edite o arquivo criado em uma etapa anterior; não crie uma segunda classe com o mesmo nome.

### ItemPedido

Crie o arquivo `src/main/java/br/edu/upf/pedidos/model/ItemPedido.java`:

```java
package br.edu.upf.pedidos.model;

import lombok.Builder;

import java.math.BigDecimal;

@Builder
public record ItemPedido(
        String produto,
        Integer quantidade,
        BigDecimal valorUnitario
) {}
```

#### Entendendo o código

> Aqui ainda não estamos falando de SOLID. Estamos apenas criando nosso modelo de domínio de forma simples.

> O próprio record elimina getters, setters e construtores escritos manualmente. Para acessar um componente, use `produto()`, `quantidade()` ou `valorUnitario()`.

> O `@Builder` implementa o padrão Builder e nos permite criar objetos de uma forma mais legível.

---

### Pedido

Crie o arquivo `src/main/java/br/edu/upf/pedidos/model/Pedido.java`:

```java
package br.edu.upf.pedidos.model;

import lombok.Builder;

import java.math.BigDecimal;
import java.util.List;

@Builder
public record Pedido(
        Long id,
        String cliente,
        List<ItemPedido> itens,
        StatusPedido status
) {

    public BigDecimal calcularTotal() {
        return itens.stream()
                .map(item -> item.valorUnitario()
                        .multiply(BigDecimal.valueOf(item.quantidade())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    public Pedido comIdEStatus(Long novoId, StatusPedido novoStatus) {
        return new Pedido(novoId, cliente, itens, novoStatus);
    }
}
```

### StatusPedido

Crie o arquivo `src/main/java/br/edu/upf/pedidos/model/StatusPedido.java`:

```java
package br.edu.upf.pedidos.model;

public enum StatusPedido {
    NOVO,
    PAGO,
    CANCELADO
}
```

---

## 4. Por que usar Builder?

Sem Builder:

Este é apenas um trecho demonstrativo. Você não precisa criar um arquivo para ele.

```java
Pedido pedido = new Pedido(
    1L,
    "Bruno",
    itens,
    StatusPedido.NOVO
);
```

### Entendendo o problema

> Esse código funciona perfeitamente.

> O problema é legibilidade. Quando olhamos apenas para `1L`, `"Bruno"`, `itens` e `StatusPedido.NOVO`, precisamos conhecer a ordem dos parâmetros do construtor.

Com Builder:

Este também é um trecho demonstrativo e usa o builder gerado para o record `Pedido`.

```java
Pedido pedido = Pedido.builder()
    .id(1L)
    .cliente("Bruno")
    .itens(itens)
    .status(StatusPedido.NOVO)
    .build();
```

### Entendendo a decisão

> Aqui o próprio código explica o objeto que está sendo criado.

> Builder não é SOLID. Estamos usando apenas como uma ferramenta para tornar o código mais legível.

> Um código mais legível reduz a quantidade de informações que precisamos guardar na cabeça enquanto programamos.

---

## 5. Primeira versão da API

Vamos começar propositalmente de uma forma simples.

Crie o arquivo `src/main/java/br/edu/upf/pedidos/controller/PedidoController.java`:

```java
package br.edu.upf.pedidos.controller;

import br.edu.upf.pedidos.model.Pedido;
import br.edu.upf.pedidos.model.StatusPedido;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.List;

@RestController
@RequestMapping("/pedidos")
public class PedidoController {

    private final List<Pedido> pedidos = new ArrayList<>();

    @PostMapping
    public Pedido criar(@RequestBody Pedido pedido) {
        Pedido novoPedido = pedido.comIdEStatus(
                (long) pedidos.size() + 1,
                StatusPedido.NOVO
        );

        pedidos.add(novoPedido);

        return novoPedido;
    }

    @GetMapping
    public List<Pedido> listar() {
        return pedidos;
    }
}
```

### Identificando o problema

> Esse código funciona. SOLID não existe porque código assim é automaticamente errado.

> O problema aparece conforme responsabilidades diferentes começam a se acumular na mesma classe.

> Vamos listar o que esse Controller está fazendo.

```text
recebe requisições HTTP
define ID
define status inicial
armazena pedidos
retorna respostas HTTP
```

> O Controller está mudando por vários motivos diferentes.

---

## 6. S — Single Responsibility Principle

### Princípio da Responsabilidade Única

> Uma classe deve possuir um único motivo para mudar.

Não confundir com:

> uma classe deve ter apenas um método.

ou:

> uma classe deve fazer literalmente apenas uma coisa.

A pergunta importante é:

> Por quais motivos esta classe pode precisar ser modificada?

No nosso Controller:

```text
Se a API mudar → Controller muda.

Se a regra de criação do pedido mudar → Controller muda.

Se a forma de armazenamento mudar → Controller muda.
```

Temos responsabilidades diferentes.

---

## 7. Aplicando SRP

Vamos separar as responsabilidades.

### PedidoService

Crie o arquivo `src/main/java/br/edu/upf/pedidos/service/PedidoService.java`:

```java
package br.edu.upf.pedidos.service;

import br.edu.upf.pedidos.model.Pedido;
import br.edu.upf.pedidos.model.StatusPedido;
import br.edu.upf.pedidos.repository.PedidoRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class PedidoService {

    private final PedidoRepository repository;

    public Pedido criar(Pedido pedido) {
        Pedido novoPedido = pedido.comIdEStatus(null, StatusPedido.NOVO);
        return repository.salvar(novoPedido);
    }

    public List<Pedido> listar() {
        return repository.listar();
    }
}
```

#### Entendendo a decisão

> Aqui aparece o **S do SOLID: Single Responsibility Principle**.

> A regra relacionada à criação de pedidos saiu do Controller e foi para o Service.

> O Service concentra regras e casos de uso relacionados ao pedido.

> Repare que não estamos separando apenas para criar mais classes. Estamos separando porque existem motivos de mudança diferentes.

---

### PedidoController

Substitua o conteúdo de `src/main/java/br/edu/upf/pedidos/controller/PedidoController.java`:

```java
package br.edu.upf.pedidos.controller;

import br.edu.upf.pedidos.model.Pedido;
import br.edu.upf.pedidos.service.PedidoService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/pedidos")
@RequiredArgsConstructor
public class PedidoController {

    private final PedidoService service;

    @PostMapping
    public Pedido criar(@RequestBody Pedido pedido) {
        return service.criar(pedido);
    }

    @GetMapping
    public List<Pedido> listar() {
        return service.listar();
    }
}
```

#### Entendendo a decisão

> O Controller agora tem uma responsabilidade muito mais clara: lidar com HTTP.

> Ele recebe a requisição, delega o trabalho e devolve a resposta.

> **Aqui também estamos aplicando o S**, porque separamos responsabilidade HTTP de regra de negócio.

---

## 8. Criando o Repository

Crie o arquivo `src/main/java/br/edu/upf/pedidos/repository/PedidoRepository.java`:

```java
package br.edu.upf.pedidos.repository;

import br.edu.upf.pedidos.model.Pedido;
import br.edu.upf.pedidos.model.StatusPedido;
import org.springframework.stereotype.Repository;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Repository
public class PedidoRepository {

    private final Map<Long, Pedido> pedidos = new HashMap<>();

    private Long proximoId = 1L;

    public Pedido salvar(Pedido pedido) {
        Pedido pedidoSalvo = pedido.comIdEStatus(
                proximoId++,
                pedido.status() == null ? StatusPedido.NOVO : pedido.status()
        );

        pedidos.put(pedidoSalvo.id(), pedidoSalvo);

        return pedidoSalvo;
    }

    public List<Pedido> listar() {
        return new ArrayList<>(pedidos.values());
    }

    public Pedido buscarPorId(Long id) {
        return pedidos.get(id);
    }
}
```

### Entendendo a decisão

> Aqui continuamos aplicando o **S do SOLID**.

> O Repository passa a ser responsável pelo armazenamento dos pedidos.

> Se amanhã mudarmos a forma de persistência, essa mudança não deveria obrigar o Controller a conhecer os detalhes.

Neste momento temos:

```text
HTTP
 ↓
PedidoController
 ↓
PedidoService
 ↓
PedidoRepository
```

---

## 9. O — Open/Closed Principle

### Princípio Aberto/Fechado

> Um código deve estar aberto para extensão e fechado para modificação.

Isso não significa que código nunca pode ser alterado.

A ideia é evitar que uma regra central precise ser constantemente modificada toda vez que surge uma nova variação de comportamento.

---

## 10. Exemplo problemático: desconto por tipo de cliente

Antes de criar as estratégias, observe este método como um trecho hipotético de `src/main/java/br/edu/upf/pedidos/service/PedidoService.java`. Não o adicione ao projeto final:

```java
public BigDecimal calcularDesconto(
        Pedido pedido,
        String tipoCliente
) {

    if (tipoCliente.equals("COMUM")) {
        return BigDecimal.ZERO;
    }

    if (tipoCliente.equals("PREMIUM")) {
        return pedido.calcularTotal()
            .multiply(new BigDecimal("0.10"));
    }

    if (tipoCliente.equals("VIP")) {
        return pedido.calcularTotal()
            .multiply(new BigDecimal("0.20"));
    }

    return BigDecimal.ZERO;
}
```

### Identificando o problema

> Este código também funciona.

> O problema é que toda vez que surge um novo tipo de desconto precisamos abrir este método e modificá-lo.

> Hoje temos COMUM, PREMIUM e VIP.

> Amanhã pode surgir FUNCIONARIO, PARCEIRO, CLUBE ou BLACK.

> O método começa a crescer e cada nova alteração aumenta a chance de quebrar algo que já funcionava.

---

## 11. Aplicando OCP

Criamos um contrato para cálculo de desconto.

Crie o arquivo `src/main/java/br/edu/upf/pedidos/desconto/PoliticaDesconto.java`:

```java
package br.edu.upf.pedidos.desconto;

import br.edu.upf.pedidos.model.Pedido;

import java.math.BigDecimal;

public interface PoliticaDesconto {

    BigDecimal calcular(Pedido pedido);
}
```

### Entendendo o contrato

> Aqui começamos a aplicar o **O do SOLID: Open/Closed Principle**.

> Em vez de concentrar todos os tipos de desconto em um grande bloco de `if`, representamos cada comportamento através de uma implementação.

---

### Cliente comum

Crie o arquivo `src/main/java/br/edu/upf/pedidos/desconto/DescontoClienteComum.java`:

```java
package br.edu.upf.pedidos.desconto;

import br.edu.upf.pedidos.model.Pedido;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;

@Component
public class DescontoClienteComum
        implements PoliticaDesconto {

    @Override
    public BigDecimal calcular(Pedido pedido) {
        return BigDecimal.ZERO;
    }
}
```

### Cliente Premium

Crie o arquivo `src/main/java/br/edu/upf/pedidos/desconto/DescontoClientePremium.java`:

```java
package br.edu.upf.pedidos.desconto;

import br.edu.upf.pedidos.model.Pedido;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;

@Component
public class DescontoClientePremium
        implements PoliticaDesconto {

    @Override
    public BigDecimal calcular(Pedido pedido) {

        return pedido.calcularTotal()
            .multiply(new BigDecimal("0.10"));
    }
}
```

### Cliente VIP

Crie o arquivo `src/main/java/br/edu/upf/pedidos/desconto/DescontoClienteVip.java`:

```java
package br.edu.upf.pedidos.desconto;

import br.edu.upf.pedidos.model.Pedido;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;

@Component
public class DescontoClienteVip
        implements PoliticaDesconto {

    @Override
    public BigDecimal calcular(Pedido pedido) {

        return pedido.calcularTotal()
            .multiply(new BigDecimal("0.20"));
    }
}
```

### Entendendo a decisão

> **Aqui temos o O do SOLID.**

> Para adicionar um novo comportamento de desconto, podemos criar uma nova implementação sem alterar as regras anteriores.

> Estamos estendendo o sistema através de novos comportamentos.

> Isso também é um exemplo simples do padrão Strategy: cada objeto representa uma estratégia diferente de cálculo.

---

## 12. L — Liskov Substitution Principle

### Princípio da Substituição de Liskov

Uma forma mais prática de entender:

> Se uma classe diz que implementa um contrato, ela deve conseguir cumprir corretamente esse contrato.

Ou ainda:

> Se meu código funciona com uma abstração, trocar uma implementação por outra não deveria quebrar as expectativas desse código.

O problema aparece quando criamos uma abstração ampla demais e algumas implementações precisam fingir que conseguem fazer algo que, na prática, não conseguem.

---

## 13. Exemplo de LSP dentro da API

Vamos imaginar que pedidos possam ser entregues de duas formas:

- entrega em endereço;
- retirada na loja.

Começamos com esta abstração:

Crie temporariamente o arquivo `src/main/java/br/edu/upf/pedidos/entrega/EntregaPedido.java`. Ele será removido ao corrigirmos o exemplo:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public interface EntregaPedido {

    BigDecimal calcularFrete();

    String buscarEnderecoEntrega();
}
```

Criamos entrega em endereço:

Crie temporariamente o arquivo `src/main/java/br/edu/upf/pedidos/entrega/EntregaEmEndereco.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public class EntregaEmEndereco
        implements EntregaPedido {

    private final String endereco;

    public EntregaEmEndereco(String endereco) {
        this.endereco = endereco;
    }

    @Override
    public BigDecimal calcularFrete() {
        return new BigDecimal("15.00");
    }

    @Override
    public String buscarEnderecoEntrega() {
        return endereco;
    }
}
```

Até aqui tudo certo.

Agora criamos retirada na loja:

Crie temporariamente o arquivo `src/main/java/br/edu/upf/pedidos/entrega/RetiradaNaLoja.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public class RetiradaNaLoja
        implements EntregaPedido {

    @Override
    public BigDecimal calcularFrete() {
        return BigDecimal.ZERO;
    }

    @Override
    public String buscarEnderecoEntrega() {
        throw new UnsupportedOperationException(
            "Retirada não possui endereço de entrega"
        );
    }
}
```

### Identificando o problema

> Aqui aparece o problema relacionado ao **L do SOLID: Liskov Substitution Principle**.

> `RetiradaNaLoja` diz que é uma `EntregaPedido`.

> Portanto, qualquer código que trabalha com `EntregaPedido` acredita que pode chamar `buscarEnderecoEntrega()`.

Veja:

Adicione temporariamente este método ao arquivo `src/main/java/br/edu/upf/pedidos/service/PedidoService.java` para observar a quebra do contrato:

```java
public void imprimirEtiqueta(EntregaPedido entrega) {

    System.out.println(
        entrega.buscarEnderecoEntrega()
    );
}
```

Podemos fazer:

Use este trecho dentro de um teste ou durante a demonstração no depurador; ele não representa um novo arquivo:

```java
EntregaPedido entrega =
    new EntregaEmEndereco("Rua A, 123");

imprimirEtiqueta(entrega);
```

Funciona.

Agora substituímos:

No mesmo teste ou depurador, substitua o trecho anterior por este:

```java
EntregaPedido entrega =
    new RetiradaNaLoja();

imprimirEtiqueta(entrega);
```

Resultado:

```text
UnsupportedOperationException
```

### Entendendo a quebra do contrato

> Esse é o ponto central do Liskov.

> Eu tinha um código trabalhando com `EntregaPedido`.

> Troquei uma implementação por outra que supostamente respeitava o mesmo contrato.

> Mas o comportamento deixou de ser válido.

> Logo, `RetiradaNaLoja` não consegue substituir corretamente `EntregaEmEndereco` dentro das expectativas estabelecidas pela interface.

> O problema normalmente não é a classe `RetiradaNaLoja`. O problema é que nossa abstração `EntregaPedido` prometeu comportamentos demais.

---

## 14. Como corrigir o problema de LSP?

Primeiro identificamos aquilo que realmente é comum às duas formas de entrega.

Exclua o conteúdo da interface temporária e substitua o arquivo `src/main/java/br/edu/upf/pedidos/entrega/EntregaPedido.java` por `src/main/java/br/edu/upf/pedidos/entrega/FormaEntrega.java` com este conteúdo:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public interface FormaEntrega {

    BigDecimal calcularFrete();
}
```

Entrega:

Substitua o conteúdo de `src/main/java/br/edu/upf/pedidos/entrega/EntregaEmEndereco.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public class EntregaEmEndereco
        implements FormaEntrega {

    private final String endereco;

    public EntregaEmEndereco(String endereco) {
        this.endereco = endereco;
    }

    @Override
    public BigDecimal calcularFrete() {
        return new BigDecimal("15.00");
    }

    public String buscarEnderecoEntrega() {
        return endereco;
    }
}
```

Retirada:

Substitua o conteúdo de `src/main/java/br/edu/upf/pedidos/entrega/RetiradaNaLoja.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public class RetiradaNaLoja
        implements FormaEntrega {

    @Override
    public BigDecimal calcularFrete() {
        return BigDecimal.ZERO;
    }
}
```

Agora:

Substitua o método temporário de `src/main/java/br/edu/upf/pedidos/service/PedidoService.java` por este:

```java
public BigDecimal calcularFrete(
        FormaEntrega formaEntrega
) {

    return formaEntrega.calcularFrete();
}
```

Podemos substituir:

No teste ou depurador usado anteriormente, execute este trecho:

```java
FormaEntrega entrega =
    new EntregaEmEndereco("Rua A");

calcularFrete(entrega);
```

ou:

Depois, substitua-o por este segundo trecho:

```java
FormaEntrega entrega =
    new RetiradaNaLoja();

calcularFrete(entrega);
```

Ambos funcionam.

### Entendendo a correção

> **Agora estamos respeitando o L do SOLID.**

> Toda implementação de `FormaEntrega` consegue cumprir o contrato que a interface promete.

> O código pode trabalhar com `FormaEntrega` sem precisar descobrir qual implementação concreta recebeu para saber se determinado método vai explodir.

> Uma boa abstração não obriga as implementações a mentir.

---

## 15. Relação entre LSP e polimorfismo

Podemos ter:

Os três trechos a seguir são exemplos de uso de `FormaEntrega` em um teste ou no depurador. Não crie arquivos para eles.

```java
FormaEntrega formaEntrega;
```

e atribuir:

No mesmo teste ou depurador, atribua a implementação de entrega em endereço:

```java
formaEntrega = new EntregaEmEndereco("Rua A");
```

ou:

Em seguida, troque somente a atribuição pela implementação de retirada:

```java
formaEntrega = new RetiradaNaLoja();
```

O código que recebe `FormaEntrega` não precisa conhecer o tipo concreto.

### Relacionando os conceitos

> Liskov é diretamente relacionado ao uso correto de polimorfismo.

> Se minhas implementações não podem realmente substituir umas às outras dentro do contrato estabelecido, meu polimorfismo está mal modelado.

---

## 16. I — Interface Segregation Principle

### Princípio da Segregação de Interfaces

> Uma classe não deveria ser obrigada a depender de operações que não utiliza.

Vamos imaginar que fizéssemos:

Considere o código abaixo como uma versão problemática e temporária de `src/main/java/br/edu/upf/pedidos/entrega/FormaEntrega.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public interface FormaEntrega {

    BigDecimal calcularFrete();

    String buscarEnderecoEntrega();

    String buscarLojaRetirada();
}
```

Entrega em endereço:

Considere esta versão temporária de `src/main/java/br/edu/upf/pedidos/entrega/EntregaEmEndereco.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public class EntregaEmEndereco
        implements FormaEntrega {

    @Override
    public BigDecimal calcularFrete() {
        return new BigDecimal("15.00");
    }

    @Override
    public String buscarEnderecoEntrega() {
        return "Rua A";
    }

    @Override
    public String buscarLojaRetirada() {
        throw new UnsupportedOperationException();
    }
}
```

Retirada:

Considere esta versão temporária de `src/main/java/br/edu/upf/pedidos/entrega/RetiradaNaLoja.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public class RetiradaNaLoja
        implements FormaEntrega {

    @Override
    public BigDecimal calcularFrete() {
        return BigDecimal.ZERO;
    }

    @Override
    public String buscarEnderecoEntrega() {
        throw new UnsupportedOperationException();
    }

    @Override
    public String buscarLojaRetirada() {
        return "Loja Centro";
    }
}
```

### Identificando o problema

> Aqui estamos vendo um problema relacionado ao **I do SOLID: Interface Segregation Principle**.

> Nossa interface ficou grande demais e começou a obrigar implementações a possuir métodos que não fazem sentido para elas.

---

## 17. Aplicando ISP

Separamos os contratos.

Mantenha `src/main/java/br/edu/upf/pedidos/entrega/FormaEntrega.java` com este conteúdo:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public interface FormaEntrega {

    BigDecimal calcularFrete();
}
```

Crie o arquivo `src/main/java/br/edu/upf/pedidos/entrega/EntregaComEndereco.java`:

```java
package br.edu.upf.pedidos.entrega;

public interface EntregaComEndereco {

    String buscarEnderecoEntrega();
}
```

Crie o arquivo `src/main/java/br/edu/upf/pedidos/entrega/RetiradaEmLoja.java`:

```java
package br.edu.upf.pedidos.entrega;

public interface RetiradaEmLoja {

    String buscarLojaRetirada();
}
```

Entrega:

Substitua o conteúdo de `src/main/java/br/edu/upf/pedidos/entrega/EntregaEmEndereco.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public class EntregaEmEndereco
        implements FormaEntrega, EntregaComEndereco {

    @Override
    public BigDecimal calcularFrete() {
        return new BigDecimal("15.00");
    }

    @Override
    public String buscarEnderecoEntrega() {
        return "Rua A";
    }
}
```

Retirada:

Substitua o conteúdo de `src/main/java/br/edu/upf/pedidos/entrega/RetiradaNaLoja.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public class RetiradaNaLoja
        implements FormaEntrega, RetiradaEmLoja {

    @Override
    public BigDecimal calcularFrete() {
        return BigDecimal.ZERO;
    }

    @Override
    public String buscarLojaRetirada() {
        return "Loja Centro";
    }
}
```

### Entendendo a decisão

> **Aqui temos o I do SOLID.**

> Cada implementação depende somente dos contratos que fazem sentido para ela.

> Interface pequena não significa necessariamente interface de um método.

> A ideia é criar contratos coesos e evitar obrigar classes a implementar operações que não possuem significado naquele contexto.

---

## 18. Diferença entre LSP e ISP

Esses dois princípios podem aparecer juntos, por isso costumam confundir.

### LSP pergunta

> Uma implementação consegue substituir outra respeitando o contrato?

Problema:

Este é o método problemático da versão temporária de `src/main/java/br/edu/upf/pedidos/entrega/RetiradaNaLoja.java`:

```java
@Override
public String buscarEnderecoEntrega() {
    throw new UnsupportedOperationException();
}
```

A implementação não cumpre corretamente aquilo que o contrato faz o código esperar.

### ISP pergunta

> Por que essa implementação foi obrigada a possuir esse método em primeiro lugar?

Por isso os princípios se relacionam.

### Relacionando os princípios

> O LSP identifica que uma implementação não consegue respeitar corretamente o contrato.

> O ISP ajuda a perceber que talvez o próprio contrato tenha responsabilidades demais e deva ser dividido.

---

## 19. D — Dependency Inversion Principle

### Princípio da Inversão de Dependência

Até agora nosso Service depende diretamente de uma classe concreta:

Este é um recorte de `src/main/java/br/edu/upf/pedidos/service/PedidoService.java`; não crie outro arquivo para ele:

```java
@Service
@RequiredArgsConstructor
public class PedidoService {

    private final PedidoRepository repository;
}
```

Temos:

```text
PedidoService
      ↓
PedidoRepository
```

### Identificando o acoplamento

> Nosso código de regra de negócio conhece diretamente uma implementação concreta de armazenamento.

> Hoje isso é memória. Amanhã poderia ser MongoDB, PostgreSQL ou outra solução.

---

## 20. Aplicando DIP

Criamos uma abstração:

Crie o arquivo `src/main/java/br/edu/upf/pedidos/repository/RepositorioPedido.java`:

```java
package br.edu.upf.pedidos.repository;

import br.edu.upf.pedidos.model.Pedido;

import java.util.List;

public interface RepositorioPedido {

    Pedido salvar(Pedido pedido);

    List<Pedido> listar();

    Pedido buscarPorId(Long id);
}
```

### Entendendo o contrato

> Aqui começamos a aplicar o **D do SOLID: Dependency Inversion Principle**.

> Nosso código de negócio passa a conhecer um contrato, e não uma tecnologia ou implementação específica.

---

Implementação em memória:

Crie o arquivo `src/main/java/br/edu/upf/pedidos/repository/PedidoRepositoryMemoria.java`:

```java
package br.edu.upf.pedidos.repository;

import br.edu.upf.pedidos.model.Pedido;
import br.edu.upf.pedidos.model.StatusPedido;
import org.springframework.stereotype.Repository;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Repository
public class PedidoRepositoryMemoria
        implements RepositorioPedido {

    private final Map<Long, Pedido> pedidos =
        new HashMap<>();

    private Long proximoId = 1L;

    @Override
    public Pedido salvar(Pedido pedido) {
        Pedido pedidoSalvo = pedido.comIdEStatus(
                proximoId++,
                pedido.status() == null ? StatusPedido.NOVO : pedido.status()
        );

        pedidos.put(pedidoSalvo.id(), pedidoSalvo);

        return pedidoSalvo;
    }

    @Override
    public List<Pedido> listar() {
        return new ArrayList<>(
            pedidos.values()
        );
    }

    @Override
    public Pedido buscarPorId(Long id) {
        return pedidos.get(id);
    }
}
```

Service:

Substitua o conteúdo de `src/main/java/br/edu/upf/pedidos/service/PedidoService.java`:

```java
package br.edu.upf.pedidos.service;

import br.edu.upf.pedidos.model.Pedido;
import br.edu.upf.pedidos.model.StatusPedido;
import br.edu.upf.pedidos.repository.RepositorioPedido;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class PedidoService {

    private final RepositorioPedido repository;

    public Pedido criar(Pedido pedido) {
        Pedido novoPedido = pedido.comIdEStatus(null, StatusPedido.NOVO);
        return repository.salvar(novoPedido);
    }

    public List<Pedido> listar() {
        return repository.listar();
    }
}
```

### Entendendo a decisão

> **Aqui temos o D do SOLID.**

> Repare que o `PedidoService` não conhece mais `PedidoRepositoryMemoria`.

> Ele conhece somente `RepositorioPedido`.

> Quem decide qual implementação será utilizada é o Spring através da injeção de dependência.

Representação:

```text
PedidoService
      ↓
RepositorioPedido
      ↑
PedidoRepositoryMemoria
```

---

## 21. DIP não é a mesma coisa que injeção de dependência

Temos:

Este é apenas um recorte de `src/main/java/br/edu/upf/pedidos/service/PedidoService.java`:

```java
private final RepositorioPedido repository;
```

e:

Esta anotação aparece na declaração da mesma classe `PedidoService`:

Confira-a em `src/main/java/br/edu/upf/pedidos/service/PedidoService.java`:

```java
@RequiredArgsConstructor
```

Spring injeta a implementação.

Isso é **Dependency Injection**.

A decisão:

```text
PedidoService
      ↓
RepositorioPedido
```

é **Dependency Inversion**.

### Diferenciando os conceitos

> Dependency Injection e Dependency Inversion são conceitos relacionados, mas não são a mesma coisa.

> Dependency Injection fala sobre como uma dependência chega até uma classe.

> Dependency Inversion fala sobre para qual direção as dependências do nosso design apontam.

---

## 22. Estrutura final do projeto

```text
src/main/java
└── br/edu/upf/pedidos
    │
    ├── controller
    │   └── PedidoController.java
    │
    ├── service
    │   └── PedidoService.java
    │
    ├── repository
    │   ├── RepositorioPedido.java
    │   └── PedidoRepositoryMemoria.java
    │
    ├── desconto
    │   ├── PoliticaDesconto.java
    │   ├── DescontoClienteComum.java
    │   ├── DescontoClientePremium.java
    │   └── DescontoClienteVip.java
    │
    ├── entrega
    │   ├── FormaEntrega.java
    │   ├── EntregaComEndereco.java
    │   ├── RetiradaEmLoja.java
    │   ├── EntregaEmEndereco.java
    │   └── RetiradaNaLoja.java
    │
    └── model
        ├── Pedido.java
        ├── ItemPedido.java
        └── StatusPedido.java
```

---

## 23. Onde está cada princípio?

### S — Single Responsibility

```text
Controller → HTTP
Service → regra de negócio
Repository → armazenamento
```

Pergunta:

> Quem é responsável por quê?

---

### O — Open/Closed

```text
PoliticaDesconto
       ↑
DescontoClienteComum
DescontoClientePremium
DescontoClienteVip
```

Podemos adicionar uma nova estratégia sem reescrever as anteriores.

Pergunta:

> Consigo adicionar um comportamento sem modificar uma regra gigante já existente?

---

### L — Liskov Substitution

```text
FormaEntrega
      ↑
EntregaEmEndereco
RetiradaNaLoja
```

Todas as implementações cumprem aquilo que `FormaEntrega` promete.

Pergunta:

> Posso substituir uma implementação por outra sem quebrar as expectativas do código?

---

### I — Interface Segregation

Em vez de:

```text
FormaEntrega
 ├── calcularFrete()
 ├── buscarEndereco()
 └── buscarLoja()
```

temos contratos específicos:

```text
FormaEntrega
EntregaComEndereco
RetiradaEmLoja
```

Pergunta:

> Estou obrigando uma classe a implementar algo que não faz sentido para ela?

---

### D — Dependency Inversion

```text
PedidoService
      ↓
RepositorioPedido
      ↑
PedidoRepositoryMemoria
```

Pergunta:

> Minha regra de negócio depende de uma implementação ou de uma abstração?

---

## 24. Exercício após SRP

Analise:

Este é um exemplo isolado de uma versão problemática de `src/main/java/br/edu/upf/pedidos/service/PedidoService.java`. Não substitua o service final por ele:

```java
@Service
public class PedidoService {

    public Pedido criar(Pedido pedido) {

        salvarPedido(pedido);

        enviarEmail(pedido);

        gerarNotaFiscal(pedido);

        return pedido;
    }
}
```

Perguntas:

1. Quantas responsabilidades diferentes existem aqui?
2. Quais motivos diferentes podem fazer essa classe mudar?
3. O que poderia ser separado?

---

## 25. Exercício após OCP

Crie uma nova política:

Crie a classe no arquivo `src/main/java/br/edu/upf/pedidos/desconto/DescontoFuncionario.java`.

```text
FUNCIONARIO
```

Regra:

```text
15% de desconto
```

Restrição:

> Não altere `DescontoClienteComum`, `DescontoClientePremium` ou `DescontoClienteVip`.

Objetivo:

Perceber que um novo comportamento pode ser adicionado através de uma nova implementação.

---

## 26. Exercício após LSP

Considere:

Para realizar o exercício, acrescente temporariamente `calcularPrazoDias()` ao arquivo `src/main/java/br/edu/upf/pedidos/entrega/FormaEntrega.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public interface FormaEntrega {

    BigDecimal calcularFrete();

    Integer calcularPrazoDias();
}
```

Uma nova modalidade chamada `RetiradaImediata` implementa:

Crie o arquivo de exercício `src/main/java/br/edu/upf/pedidos/entrega/RetiradaImediata.java`:

```java
package br.edu.upf.pedidos.entrega;

import java.math.BigDecimal;

public class RetiradaImediata implements FormaEntrega {

    @Override
    public BigDecimal calcularFrete() {
        return BigDecimal.ZERO;
    }

    @Override
    public Integer calcularPrazoDias() {
        throw new UnsupportedOperationException();
    }
}
```

Perguntas:

1. Essa implementação respeita o contrato?
2. Um código que trabalha apenas com `FormaEntrega` pode confiar nessa implementação?
3. Existe um problema de modelagem na abstração?

---

## 27. Exercício após ISP

Considere:

Crie o arquivo de exercício `src/main/java/br/edu/upf/pedidos/notificacao/Notificacao.java`:

```java
package br.edu.upf.pedidos.notificacao;

public interface Notificacao {

    void enviarEmail();

    void enviarSms();

    void enviarWhatsApp();
}
```

Uma implementação suporta apenas e-mail.

Pergunta:

> Ela deveria ser obrigada a implementar os três métodos?

Proponha uma divisão melhor dos contratos.

---

## 28. Exercício após DIP

Crie:

Crie o arquivo de exercício `src/test/java/br/edu/upf/pedidos/repository/PedidoRepositoryFake.java`:

```java
package br.edu.upf.pedidos.repository;

import br.edu.upf.pedidos.model.Pedido;

import java.util.List;

public class PedidoRepositoryFake
        implements RepositorioPedido {

    @Override
    public Pedido salvar(Pedido pedido) {
        return pedido;
    }

    @Override
    public List<Pedido> listar() {
        return List.of();
    }

    @Override
    public Pedido buscarPorId(Long id) {
        return null;
    }
}
```

Implemente os métodos retornando dados fixos.

Depois utilize essa implementação sem modificar a lógica de negócio de `PedidoService`.

Objetivo:

Perceber que o Service depende do contrato e não da tecnologia de armazenamento.

---

## 29. Erros comuns

### Criar uma interface para cada classe

SOLID não exige que toda classe possua uma interface correspondente. Crie uma abstração quando houver mais de uma implementação, quando for necessário proteger uma regra de negócio de um detalhe externo ou quando uma mudança concreta justificar essa separação.

### Confundir responsabilidade única com classe de um método

Uma classe pode ter vários métodos relacionados à mesma responsabilidade. O problema aparece quando ela reúne motivos de mudança diferentes, como tratar HTTP, aplicar regras de negócio e armazenar dados.

### Aplicar todos os princípios ao mesmo tempo

Durante a prática, cada princípio responde a um problema específico. Antes de criar uma nova classe ou interface, identifique qual dificuldade ela resolve. Se não houver uma resposta concreta, a abstração provavelmente ainda não é necessária.

### Confundir DIP com injeção de dependência

A injeção de dependência é o mecanismo usado para entregar uma dependência a uma classe. O DIP orienta o desenho das dependências: regras centrais devem depender de contratos adequados, e não de detalhes concretos que mudam com frequência.

---

## 30. Resumo

```text
S — Quem é responsável por quê?

O — Consigo adicionar comportamento sem modificar regras existentes?

L — Minhas implementações realmente cumprem seus contratos?

I — Estou obrigando alguém a depender do que não utiliza?

D — Minha regra depende de uma implementação ou de uma abstração?
```

---

### Conclusão

> SOLID não é uma arquitetura.

> SOLID não significa criar interface para tudo.

> SOLID não significa criar dezenas de classes.

> SOLID é um conjunto de princípios que ajuda a organizar responsabilidades, contratos e dependências.

O objetivo não é escrever o código mais sofisticado possível.

O objetivo é escrever um código que continue simples mesmo quando o sistema crescer.

---

## Roteiro da prática

Em uma aula de aproximadamente quatro horas, você alternará explicações curtas com implementação. A maior parte do tempo será dedicada a modificar e analisar o código:

```text
30% explicação
70% live coding
```

Siga esta ordem:

```text
1. Criar projeto Spring
2. Criar modelos
3. Mostrar Lombok e Builder
4. Criar Controller simples
5. Identificar responsabilidades
6. Aplicar SRP
7. Criar desconto inicialmente com if
8. Aplicar OCP
9. Criar exemplo de forma de entrega
10. Quebrar o contrato propositalmente
11. Explicar LSP
12. Perceber interface grande demais
13. Aplicar ISP
14. Criar abstração do Repository
15. Aplicar DIP
16. Revisar o projeto identificando S, O, L, I e D
```

Em cada princípio, siga o mesmo raciocínio:

> primeiro criar ou mostrar o problema;

> depois perguntar o que está incomodando no código;

> somente então apresentar o princípio SOLID correspondente.

Assim, cada princípio aparece como resposta a um problema que você observou no código, e não como uma definição isolada para decorar.
