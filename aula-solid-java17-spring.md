# Aula — SOLID com Java 17 e Spring Boot

## Tema

Aplicando princípios SOLID durante a construção de uma API simples de pedidos com Java 17, Spring Boot e Lombok.

---

## Objetivo da aula

Ao final desta aula, o aluno deve conseguir:

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

# 1. Projeto da aula

Vamos construir uma API extremamente simples de pedidos.

## Stack

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

# 2. Domínio

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

# 3. Criando os modelos com Lombok

## ItemPedido

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ItemPedido {

    private String produto;
    private Integer quantidade;
    private BigDecimal valorUnitario;
}
```

### O que comentar em aula

> Aqui ainda não estamos falando de SOLID. Estamos apenas criando nosso modelo de domínio de forma simples.

> O Lombok reduz código repetitivo. Em vez de escrever manualmente getters, setters e construtores, usamos anotações que fazem essa geração durante a compilação.

> O `@Builder` implementa o padrão Builder e nos permite criar objetos de uma forma mais legível.

---

## Pedido

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Pedido {

    private Long id;
    private String cliente;
    private List<ItemPedido> itens;
    private StatusPedido status;
}
```

## StatusPedido

```java
public enum StatusPedido {
    NOVO,
    PAGO,
    CANCELADO
}
```

---

# 4. Por que usar Builder?

Sem Builder:

```java
Pedido pedido = new Pedido(
    1L,
    "Bruno",
    itens,
    StatusPedido.NOVO
);
```

### O que comentar em aula

> Esse código funciona perfeitamente.

> O problema é legibilidade. Quando olhamos apenas para `1L`, `"Bruno"`, `itens` e `StatusPedido.NOVO`, precisamos conhecer a ordem dos parâmetros do construtor.

Com Builder:

```java
Pedido pedido = Pedido.builder()
    .id(1L)
    .cliente("Bruno")
    .itens(itens)
    .status(StatusPedido.NOVO)
    .build();
```

### O que comentar em aula

> Aqui o próprio código explica o objeto que está sendo criado.

> Builder não é SOLID. Estamos usando apenas como uma ferramenta para tornar o código mais legível.

> Um código mais legível reduz a quantidade de informações que precisamos guardar na cabeça enquanto programamos.

---

# 5. Primeira versão da API

Vamos começar propositalmente de uma forma simples.

```java
@RestController
@RequestMapping("/pedidos")
public class PedidoController {

    private final List<Pedido> pedidos = new ArrayList<>();

    @PostMapping
    public Pedido criar(@RequestBody Pedido pedido) {

        pedido.setId((long) pedidos.size() + 1);
        pedido.setStatus(StatusPedido.NOVO);

        pedidos.add(pedido);

        return pedido;
    }

    @GetMapping
    public List<Pedido> listar() {
        return pedidos;
    }
}
```

### O que comentar em aula

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

# 6. S — Single Responsibility Principle

## Princípio da Responsabilidade Única

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

# 7. Aplicando SRP

Vamos separar as responsabilidades.

## PedidoService

```java
@Service
@RequiredArgsConstructor
public class PedidoService {

    private final PedidoRepository repository;

    public Pedido criar(Pedido pedido) {

        pedido.setStatus(StatusPedido.NOVO);

        return repository.salvar(pedido);
    }

    public List<Pedido> listar() {
        return repository.listar();
    }
}
```

### O que comentar em aula

> Aqui aparece o **S do SOLID: Single Responsibility Principle**.

> A regra relacionada à criação de pedidos saiu do Controller e foi para o Service.

> O Service concentra regras e casos de uso relacionados ao pedido.

> Repare que não estamos separando apenas para criar mais classes. Estamos separando porque existem motivos de mudança diferentes.

---

## PedidoController

```java
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

### O que comentar em aula

> O Controller agora tem uma responsabilidade muito mais clara: lidar com HTTP.

> Ele recebe a requisição, delega o trabalho e devolve a resposta.

> **Aqui também estamos aplicando o S**, porque separamos responsabilidade HTTP de regra de negócio.

---

# 8. Criando o Repository

```java
@Repository
public class PedidoRepository {

    private final Map<Long, Pedido> pedidos = new HashMap<>();

    private Long proximoId = 1L;

    public Pedido salvar(Pedido pedido) {

        pedido.setId(proximoId++);

        pedidos.put(pedido.getId(), pedido);

        return pedido;
    }

    public List<Pedido> listar() {
        return new ArrayList<>(pedidos.values());
    }

    public Optional<Pedido> buscarPorId(Long id) {
        return Optional.ofNullable(pedidos.get(id));
    }
}
```

### O que comentar em aula

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

# 9. O — Open/Closed Principle

## Princípio Aberto/Fechado

> Um código deve estar aberto para extensão e fechado para modificação.

Isso não significa que código nunca pode ser alterado.

A ideia é evitar que uma regra central precise ser constantemente modificada toda vez que surge uma nova variação de comportamento.

---

# 10. Exemplo ruim: desconto por tipo de cliente

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

### O que comentar em aula

> Este código também funciona.

> O problema é que toda vez que surge um novo tipo de desconto precisamos abrir este método e modificá-lo.

> Hoje temos COMUM, PREMIUM e VIP.

> Amanhã pode surgir FUNCIONARIO, PARCEIRO, CLUBE ou BLACK.

> O método começa a crescer e cada nova alteração aumenta a chance de quebrar algo que já funcionava.

---

# 11. Aplicando OCP

Criamos um contrato para cálculo de desconto.

```java
public interface PoliticaDesconto {

    BigDecimal calcular(Pedido pedido);
}
```

### O que comentar em aula

> Aqui começamos a aplicar o **O do SOLID: Open/Closed Principle**.

> Em vez de concentrar todos os tipos de desconto em um grande bloco de `if`, representamos cada comportamento através de uma implementação.

---

## Cliente comum

```java
@Component
public class DescontoClienteComum
        implements PoliticaDesconto {

    @Override
    public BigDecimal calcular(Pedido pedido) {
        return BigDecimal.ZERO;
    }
}
```

## Cliente Premium

```java
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

## Cliente VIP

```java
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

### O que comentar em aula

> **Aqui temos o O do SOLID.**

> Para adicionar um novo comportamento de desconto, podemos criar uma nova implementação sem alterar as regras anteriores.

> Estamos estendendo o sistema através de novos comportamentos.

> Isso também é um exemplo simples do padrão Strategy: cada objeto representa uma estratégia diferente de cálculo.

---

# 12. L — Liskov Substitution Principle

## Princípio da Substituição de Liskov

Uma forma mais prática de entender:

> Se uma classe diz que implementa um contrato, ela deve conseguir cumprir corretamente esse contrato.

Ou ainda:

> Se meu código funciona com uma abstração, trocar uma implementação por outra não deveria quebrar as expectativas desse código.

O problema aparece quando criamos uma abstração ampla demais e algumas implementações precisam fingir que conseguem fazer algo que, na prática, não conseguem.

---

# 13. Exemplo de LSP dentro da nossa API

Vamos imaginar que pedidos possam ser entregues de duas formas:

- entrega em endereço;
- retirada na loja.

Começamos com esta abstração:

```java
public interface EntregaPedido {

    BigDecimal calcularFrete();

    String buscarEnderecoEntrega();
}
```

Criamos entrega em endereço:

```java
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

```java
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

### O que comentar em aula

> Aqui aparece o problema relacionado ao **L do SOLID: Liskov Substitution Principle**.

> `RetiradaNaLoja` diz que é uma `EntregaPedido`.

> Portanto, qualquer código que trabalha com `EntregaPedido` acredita que pode chamar `buscarEnderecoEntrega()`.

Veja:

```java
public void imprimirEtiqueta(EntregaPedido entrega) {

    System.out.println(
        entrega.buscarEnderecoEntrega()
    );
}
```

Podemos fazer:

```java
EntregaPedido entrega =
    new EntregaEmEndereco("Rua A, 123");

imprimirEtiqueta(entrega);
```

Funciona.

Agora substituímos:

```java
EntregaPedido entrega =
    new RetiradaNaLoja();

imprimirEtiqueta(entrega);
```

Resultado:

```text
UnsupportedOperationException
```

### O que comentar em aula

> Esse é o ponto central do Liskov.

> Eu tinha um código trabalhando com `EntregaPedido`.

> Troquei uma implementação por outra que supostamente respeitava o mesmo contrato.

> Mas o comportamento deixou de ser válido.

> Logo, `RetiradaNaLoja` não consegue substituir corretamente `EntregaEmEndereco` dentro das expectativas estabelecidas pela interface.

> O problema normalmente não é a classe `RetiradaNaLoja`. O problema é que nossa abstração `EntregaPedido` prometeu comportamentos demais.

---

# 14. Como corrigimos o problema de LSP?

Primeiro identificamos aquilo que realmente é comum às duas formas de entrega.

```java
public interface FormaEntrega {

    BigDecimal calcularFrete();
}
```

Entrega:

```java
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

```java
public class RetiradaNaLoja
        implements FormaEntrega {

    @Override
    public BigDecimal calcularFrete() {
        return BigDecimal.ZERO;
    }
}
```

Agora:

```java
public BigDecimal calcularFrete(
        FormaEntrega formaEntrega
) {

    return formaEntrega.calcularFrete();
}
```

Podemos substituir:

```java
FormaEntrega entrega =
    new EntregaEmEndereco("Rua A");

calcularFrete(entrega);
```

ou:

```java
FormaEntrega entrega =
    new RetiradaNaLoja();

calcularFrete(entrega);
```

Ambos funcionam.

### O que comentar em aula

> **Agora estamos respeitando o L do SOLID.**

> Toda implementação de `FormaEntrega` consegue cumprir o contrato que a interface promete.

> O código pode trabalhar com `FormaEntrega` sem precisar descobrir qual implementação concreta recebeu para saber se determinado método vai explodir.

> Uma boa abstração não obriga as implementações a mentir.

---

# 15. Relação entre LSP e polimorfismo

Podemos ter:

```java
FormaEntrega formaEntrega;
```

e atribuir:

```java
formaEntrega = new EntregaEmEndereco("Rua A");
```

ou:

```java
formaEntrega = new RetiradaNaLoja();
```

O código que recebe `FormaEntrega` não precisa conhecer o tipo concreto.

### O que comentar em aula

> Liskov é diretamente relacionado ao uso correto de polimorfismo.

> Se minhas implementações não podem realmente substituir umas às outras dentro do contrato estabelecido, meu polimorfismo está mal modelado.

---

# 16. I — Interface Segregation Principle

## Princípio da Segregação de Interfaces

> Uma classe não deveria ser obrigada a depender de operações que não utiliza.

Vamos imaginar que fizéssemos:

```java
public interface FormaEntrega {

    BigDecimal calcularFrete();

    String buscarEnderecoEntrega();

    String buscarLojaRetirada();
}
```

Entrega em endereço:

```java
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

```java
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

### O que comentar em aula

> Aqui estamos vendo um problema relacionado ao **I do SOLID: Interface Segregation Principle**.

> Nossa interface ficou grande demais e começou a obrigar implementações a possuir métodos que não fazem sentido para elas.

---

# 17. Aplicando ISP

Separamos os contratos.

```java
public interface FormaEntrega {

    BigDecimal calcularFrete();
}
```

```java
public interface EntregaComEndereco {

    String buscarEnderecoEntrega();
}
```

```java
public interface RetiradaEmLoja {

    String buscarLojaRetirada();
}
```

Entrega:

```java
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

```java
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

### O que comentar em aula

> **Aqui temos o I do SOLID.**

> Cada implementação depende somente dos contratos que fazem sentido para ela.

> Interface pequena não significa necessariamente interface de um método.

> A ideia é criar contratos coesos e evitar obrigar classes a implementar operações que não possuem significado naquele contexto.

---

# 18. Diferença entre LSP e ISP

Esses dois princípios podem aparecer juntos, por isso costumam confundir.

## LSP pergunta:

> Uma implementação consegue substituir outra respeitando o contrato?

Problema:

```java
@Override
public String buscarEnderecoEntrega() {
    throw new UnsupportedOperationException();
}
```

A implementação não cumpre corretamente aquilo que o contrato faz o código esperar.

## ISP pergunta:

> Por que essa implementação foi obrigada a possuir esse método em primeiro lugar?

Por isso os princípios se relacionam.

### O que comentar em aula

> O LSP identifica que uma implementação não consegue respeitar corretamente o contrato.

> O ISP ajuda a perceber que talvez o próprio contrato tenha responsabilidades demais e deva ser dividido.

---

# 19. D — Dependency Inversion Principle

## Princípio da Inversão de Dependência

Até agora nosso Service depende diretamente de uma classe concreta:

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

### O que comentar em aula

> Nosso código de regra de negócio conhece diretamente uma implementação concreta de armazenamento.

> Hoje isso é memória. Amanhã poderia ser MongoDB, PostgreSQL ou outra solução.

---

# 20. Aplicando DIP

Criamos uma abstração:

```java
public interface RepositorioPedido {

    Pedido salvar(Pedido pedido);

    List<Pedido> listar();

    Optional<Pedido> buscarPorId(Long id);
}
```

### O que comentar em aula

> Aqui começamos a aplicar o **D do SOLID: Dependency Inversion Principle**.

> Nosso código de negócio passa a conhecer um contrato, e não uma tecnologia ou implementação específica.

---

Implementação em memória:

```java
@Repository
public class PedidoRepositoryMemoria
        implements RepositorioPedido {

    private final Map<Long, Pedido> pedidos =
        new HashMap<>();

    private Long proximoId = 1L;

    @Override
    public Pedido salvar(Pedido pedido) {

        pedido.setId(proximoId++);

        pedidos.put(
            pedido.getId(),
            pedido
        );

        return pedido;
    }

    @Override
    public List<Pedido> listar() {
        return new ArrayList<>(
            pedidos.values()
        );
    }

    @Override
    public Optional<Pedido> buscarPorId(Long id) {
        return Optional.ofNullable(
            pedidos.get(id)
        );
    }
}
```

Service:

```java
@Service
@RequiredArgsConstructor
public class PedidoService {

    private final RepositorioPedido repository;

    public Pedido criar(Pedido pedido) {

        pedido.setStatus(StatusPedido.NOVO);

        return repository.salvar(pedido);
    }

    public List<Pedido> listar() {
        return repository.listar();
    }
}
```

### O que comentar em aula

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

# 21. DIP não é a mesma coisa que Dependency Injection

Temos:

```java
private final RepositorioPedido repository;
```

e:

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

### O que comentar em aula

> Dependency Injection e Dependency Inversion são conceitos relacionados, mas não são a mesma coisa.

> Dependency Injection fala sobre como uma dependência chega até uma classe.

> Dependency Inversion fala sobre para qual direção as dependências do nosso design apontam.

---

# 22. Estrutura final do projeto

```text
src/main/java
└── com.exemplo.pedidos
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

# 23. Onde está cada princípio?

## S — Single Responsibility

```text
Controller → HTTP
Service → regra de negócio
Repository → armazenamento
```

Pergunta:

> Quem é responsável por quê?

---

## O — Open/Closed

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

## L — Liskov Substitution

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

## I — Interface Segregation

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

## D — Dependency Inversion

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

# 24. Exercício após SRP

Analise:

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

# 25. Exercício após OCP

Crie uma nova política:

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

# 26. Exercício após LSP

Considere:

```java
public interface FormaEntrega {

    BigDecimal calcularFrete();

    Integer calcularPrazoDias();
}
```

Uma nova modalidade chamada `RetiradaImediata` implementa:

```java
@Override
public Integer calcularPrazoDias() {
    throw new UnsupportedOperationException();
}
```

Perguntas:

1. Essa implementação respeita o contrato?
2. Um código que trabalha apenas com `FormaEntrega` pode confiar nessa implementação?
3. Existe um problema de modelagem na abstração?

---

# 27. Exercício após ISP

Considere:

```java
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

# 28. Exercício após DIP

Crie:

```java
public class PedidoRepositoryFake
        implements RepositorioPedido {
}
```

Implemente os métodos retornando dados fixos.

Depois utilize essa implementação sem modificar a lógica de negócio de `PedidoService`.

Objetivo:

Perceber que o Service depende do contrato e não da tecnologia de armazenamento.

---

# 29. Resumo final

```text
S — Quem é responsável por quê?

O — Consigo adicionar comportamento sem modificar regras existentes?

L — Minhas implementações realmente cumprem seus contratos?

I — Estou obrigando alguém a depender do que não utiliza?

D — Minha regra depende de uma implementação ou de uma abstração?
```

---

# 30. Mensagem final

> SOLID não é uma arquitetura.

> SOLID não significa criar interface para tudo.

> SOLID não significa criar dezenas de classes.

> SOLID é um conjunto de princípios que ajuda a organizar responsabilidades, contratos e dependências.

O objetivo não é escrever o código mais sofisticado possível.

O objetivo é escrever um código que continue simples mesmo quando o sistema crescer.

---

# Sugestão de dinâmica da aula

Para uma aula de aproximadamente 4 horas:

```text
30% explicação
70% live coding
```

Ordem recomendada:

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

A principal estratégia didática da aula deve ser:

> primeiro criar ou mostrar o problema;

> depois perguntar o que está incomodando no código;

> somente então apresentar o princípio SOLID correspondente.

Isso faz o princípio aparecer como solução de um problema real, e não como uma definição decorada.
