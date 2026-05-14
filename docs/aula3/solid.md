# Princípios SOLID em Java 17 com Spring Boot

## Objetivo da aula

Nesta aula vamos entender os cinco princípios SOLID usando Java 17.

A ideia não é decorar siglas. A ideia é olhar para um código de pagamento que funciona, mas ficou difícil de manter, e depois melhorar esse código aos poucos.

Vamos usar o mesmo domínio do tutorial da Payment API:

- pagamento por PIX
- pagamento por cartão de crédito
- pagamento por boleto
- persistência em MongoDB
- API REST com Spring Boot

## Resultado final

Ao final da aula, você vai conseguir identificar problemas como:

- classe fazendo coisa demais
- muitos `if` e `else` para escolher comportamento
- dependência direta de implementação concreta
- interfaces grandes demais
- código difícil de testar

E vai conseguir melhorar isso usando:

- services menores
- interfaces simples
- strategy pattern
- injeção de dependência pelo construtor
- Spring Data MongoDB separado da regra de negócio

## Contexto

Em uma API real, é comum começar com uma classe simples:

```java
public Payment createPayment(CreatePaymentRequest request) {
    // valida
    // escolhe tipo de pagamento
    // calcula taxa
    // salva no banco
    // envia notificacao
    // monta resposta
}
```

No começo isso parece prático. O problema aparece quando o sistema cresce.

Imagine que agora o negócio precisa adicionar débito, PayPal, criptomoeda, estorno, auditoria e antifraude. Se tudo estiver dentro da mesma classe, cada mudança encosta em código antigo. Isso aumenta a chance de quebrar algo que já estava funcionando.

SOLID ajuda justamente nesse ponto: criar código que aceita mudança com menos dor.

## Explicação conceitual

SOLID é um conjunto de cinco princípios de programação orientada a objetos:

- **S**: Single Responsibility Principle
- **O**: Open/Closed Principle
- **L**: Liskov Substitution Principle
- **I**: Interface Segregation Principle
- **D**: Dependency Inversion Principle

Eles não obrigam você a criar arquitetura sofisticada. Em uma API Spring Boot, muitas vezes SOLID aparece em decisões simples:

- controller só recebe HTTP e delega
- service concentra regra de negócio
- repository cuida do MongoDB
- DTO separa entrada e saída da API
- interface aparece quando existe mais de uma implementação real

## Setup inicial

Os exemplos desta aula usam Java 17 e Spring Boot. O projeto prático do tutorial usa:

- Java 17
- Gradle
- Spring Web
- Spring Data MongoDB
- Bean Validation
- Springdoc OpenAPI
- Docker para subir o MongoDB

## Passo a passo

### 1. Single Responsibility Principle

O princípio da responsabilidade única diz:

> Uma classe deve ter apenas uma razão para mudar.

Isso não significa que uma classe só pode ter um método. Significa que ela deve cuidar de um assunto só.

#### Código legado sem SRP

```java
package br.edu.upf.paymentapi.legacy;

import br.edu.upf.paymentapi.model.Payment;
import br.edu.upf.paymentapi.repository.PaymentRepository;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

import static org.springframework.http.HttpStatus.BAD_REQUEST;

@Service
public class LegacyPaymentService {

    private final PaymentRepository paymentRepository;

    public LegacyPaymentService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    public Payment createPayment(CreateLegacyPaymentRequest request) {
        if (request.customerName() == null || request.customerName().isBlank()) {
            throw new ResponseStatusException(BAD_REQUEST, "Nome do cliente e obrigatorio");
        }

        if (request.amount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new ResponseStatusException(BAD_REQUEST, "Valor deve ser maior que zero");
        }

        Map<String, Object> metadata = new HashMap<>();

        if ("PIX".equals(request.type())) {
            metadata.put("qrCode", "pix-" + System.currentTimeMillis());
            metadata.put("pixKey", request.pixKey());
        } else if ("CREDIT_CARD".equals(request.type())) {
            metadata.put("cardNumber", maskCard(request.cardNumber()));
            metadata.put("processor", detectCardProcessor(request.cardNumber()));
        } else if ("BOLETO".equals(request.type())) {
            metadata.put("boletoNumber", "34191.79001 01043.510047 91020.150008 1");
            metadata.put("dueDate", request.dueDate());
        } else {
            throw new ResponseStatusException(BAD_REQUEST, "Tipo de pagamento nao suportado");
        }

        BigDecimal fee = calculateFee(request.amount(), request.type());
        String status = determineStatus(request.amount(), request.type());

        Payment payment = new Payment();
        payment.setType(request.type());
        payment.setAmount(request.amount().add(fee));
        payment.setStatus(status);
        payment.setCustomerName(request.customerName().toUpperCase());
        payment.setCustomerEmail(request.customerEmail().toLowerCase());
        payment.setMetadata(metadata);
        payment.setCreatedAt(LocalDateTime.now());
        payment.setUpdatedAt(LocalDateTime.now());

        Payment savedPayment = paymentRepository.save(payment);

        System.out.println("Email enviado para " + savedPayment.getCustomerEmail());
        System.out.println("Pagamento criado: " + savedPayment.getId());

        return savedPayment;
    }

    private BigDecimal calculateFee(BigDecimal amount, String type) {
        if ("CREDIT_CARD".equals(type)) {
            return amount.multiply(new BigDecimal("0.03"));
        }
        if ("BOLETO".equals(type)) {
            return new BigDecimal("2.50");
        }
        return BigDecimal.ZERO;
    }

    private String determineStatus(BigDecimal amount, String type) {
        if (amount.compareTo(new BigDecimal("10000")) > 0) {
            return "PENDING";
        }
        return "APPROVED";
    }

    private String maskCard(String cardNumber) {
        return "************" + cardNumber.substring(cardNumber.length() - 4);
    }

    private String detectCardProcessor(String cardNumber) {
        if (cardNumber.startsWith("4")) {
            return "Visa";
        }
        if (cardNumber.startsWith("5")) {
            return "Mastercard";
        }
        return "Unknown";
    }
}
```

Esse service faz muitas coisas:

- valida dados
- processa PIX, cartão e boleto
- calcula taxa
- decide status
- monta documento para o MongoDB
- salva no banco
- simula envio de email
- escreve log no console

Se mudar a regra de taxa, mexemos nessa classe. Se mudar o envio de email, mexemos nessa classe. Se mudar o MongoDB, mexemos nessa classe. São muitas razões para mudar.

#### Código melhor com SRP

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import br.edu.upf.paymentapi.model.Payment;
import br.edu.upf.paymentapi.repository.PaymentRepository;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import org.springframework.stereotype.Service;

@Service
public class PaymentService {

    private final PaymentProcessorResolver processorResolver;
    private final PaymentRepository paymentRepository;

    public PaymentService(
            PaymentProcessorResolver processorResolver,
            PaymentRepository paymentRepository
    ) {
        this.processorResolver = processorResolver;
        this.paymentRepository = paymentRepository;
    }

    public Payment createPayment(CreatePaymentRequest request) {
        PaymentProcessor processor = processorResolver.resolve(request.type());
        ProcessedPayment processedPayment = processor.process(request);

        Payment payment = new Payment();
        payment.setType(request.type());
        payment.setAmount(request.amount().add(processor.calculateFee(request.amount())));
        payment.setStatus(determineStatus(request.amount()));
        payment.setCustomerName(request.customerName());
        payment.setCustomerEmail(request.customerEmail());
        payment.setMetadata(processedPayment.metadata());
        payment.setCreatedAt(LocalDateTime.now());
        payment.setUpdatedAt(LocalDateTime.now());

        return paymentRepository.save(payment);
    }

    private String determineStatus(BigDecimal amount) {
        if (amount.compareTo(new BigDecimal("10000")) > 0) {
            return "PENDING";
        }
        return "APPROVED";
    }
}
```

Agora o service principal orquestra o caso de uso. Ele não sabe os detalhes de como gerar QR Code de PIX ou mascarar cartão.

### 2. Open/Closed Principle

O princípio aberto-fechado diz:

> Aberto para extensão, fechado para modificação.

Na prática, quando entra um novo tipo de pagamento, o ideal é adicionar uma nova classe, não aumentar um `if` gigante dentro do service principal.

#### Código legado sem OCP

```java
if ("PIX".equals(request.type())) {
    // processa PIX
} else if ("CREDIT_CARD".equals(request.type())) {
    // processa cartao
} else if ("BOLETO".equals(request.type())) {
    // processa boleto
} else if ("PAYPAL".equals(request.type())) {
    // processa PayPal
} else if ("CRYPTO".equals(request.type())) {
    // processa cripto
}
```

Esse código cresce toda vez que aparece um novo meio de pagamento.

#### Código melhor com Strategy

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import java.math.BigDecimal;

public interface PaymentProcessor {

    String getType();

    ProcessedPayment process(CreatePaymentRequest request);

    BigDecimal calculateFee(BigDecimal amount);
}
```

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import java.math.BigDecimal;
import java.util.Map;
import org.springframework.stereotype.Service;

@Service
public class PixPaymentProcessor implements PaymentProcessor {

    @Override
    public String getType() {
        return "PIX";
    }

    @Override
    public ProcessedPayment process(CreatePaymentRequest request) {
        return new ProcessedPayment(Map.of(
                "pixKey", request.pixKey(),
                "qrCode", "pix-" + System.currentTimeMillis()
        ));
    }

    @Override
    public BigDecimal calculateFee(BigDecimal amount) {
        return BigDecimal.ZERO;
    }
}
```

```java
package br.edu.upf.paymentapi.service;

import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ResponseStatusException;

@Component
public class PaymentProcessorResolver {

    private final Map<String, PaymentProcessor> processors;

    public PaymentProcessorResolver(List<PaymentProcessor> processors) {
        this.processors = processors.stream()
                .collect(Collectors.toMap(PaymentProcessor::getType, Function.identity()));
    }

    public PaymentProcessor resolve(String type) {
        PaymentProcessor processor = processors.get(type);

        if (processor == null) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Tipo de pagamento nao suportado");
        }

        return processor;
    }
}
```

O Spring injeta todos os beans que implementam `PaymentProcessor`. O resolver monta um `Map` usando o tipo de cada processador.

Para adicionar débito, por exemplo, criamos `DebitCardPaymentProcessor`. O service principal não precisa mudar.

### 3. Liskov Substitution Principle

O princípio da substituição de Liskov diz:

> Uma implementação deve poder substituir outra sem quebrar o contrato esperado.

Se `PixPaymentProcessor`, `CreditCardPaymentProcessor` e `BoletoPaymentProcessor` implementam `PaymentProcessor`, o `PaymentService` deve conseguir usar qualquer um deles do mesmo jeito.

#### Código problemático

```java
public class CreditCardPaymentProcessor implements PaymentProcessor {

    @Override
    public ProcessedPayment process(CreatePaymentRequest request) {
        if (request.amount().compareTo(new BigDecimal("10000")) > 0) {
            throw new IllegalStateException("Cartao nao aceita valor alto");
        }

        return new ProcessedPayment(Map.of("processor", "Visa"));
    }
}
```

O problema não é lançar exceção. O problema é o contrato ficar confuso. Se algumas implementações retornam resultado e outras interrompem o fluxo por regra de negócio previsível, o código cliente precisa conhecer detalhes de cada implementação.

#### Código melhor

```java
package br.edu.upf.paymentapi.service;

public record PaymentProcessResult(
        boolean approved,
        String reason,
        ProcessedPayment processedPayment
) {
}
```

```java
public interface PaymentProcessor {

    String getType();

    PaymentProcessResult process(CreatePaymentRequest request);

    BigDecimal calculateFee(BigDecimal amount);
}
```

```java
@Override
public PaymentProcessResult process(CreatePaymentRequest request) {
    if (request.amount().compareTo(new BigDecimal("10000")) > 0) {
        return new PaymentProcessResult(false, "Valor acima do limite do cartao", null);
    }

    ProcessedPayment processedPayment = new ProcessedPayment(Map.of(
            "cardNumber", maskCard(request.cardNumber()),
            "processor", detectProcessor(request.cardNumber())
    ));

    return new PaymentProcessResult(true, null, processedPayment);
}
```

Agora todas as implementações obedecem ao mesmo contrato: processar sempre devolve um resultado. O service decide o que fazer com esse resultado.

### 4. Interface Segregation Principle

O princípio de segregação de interfaces diz:

> Um cliente não deve depender de métodos que ele não usa.

Em Java, isso aparece quando criamos uma interface grande demais.

#### Interface ruim

```java
public interface PaymentOperations {

    Payment create(CreatePaymentRequest request);

    Payment findById(String id);

    List<Payment> findAll();

    void sendEmail(Payment payment);

    void sendSms(Payment payment);

    BigDecimal calculateFee(BigDecimal amount, String type);

    byte[] exportToPdf();

    byte[] exportToExcel();
}
```

Quem só precisa calcular taxa acaba dependendo de exportação PDF, SMS e busca no banco.

#### Interfaces menores

```java
public interface PaymentProcessor {

    String getType();

    PaymentProcessResult process(CreatePaymentRequest request);

    BigDecimal calculateFee(BigDecimal amount);
}
```

```java
public interface PaymentNotifier {

    void notifyCreated(Payment payment);
}
```

```java
public interface PaymentReportExporter {

    byte[] export();
}
```

Interfaces pequenas deixam claro o papel de cada componente.

### 5. Dependency Inversion Principle

O princípio da inversão de dependência diz:

> Dependa de abstrações, não de implementações concretas.

No Spring, isso aparece bastante com injeção de dependência.

#### Código acoplado

```java
public class PaymentService {

    private final PaymentRepository paymentRepository = new PaymentRepository();
    private final EmailPaymentNotifier notifier = new EmailPaymentNotifier();
}
```

Esse exemplo nem combina com Spring Data, porque repository do Spring é uma interface gerenciada pelo container. Além disso, criar dependências com `new` dentro da classe dificulta testes.

#### Código melhor

```java
@Service
public class PaymentService {

    private final PaymentRepository paymentRepository;
    private final PaymentNotifier notifier;

    public PaymentService(PaymentRepository paymentRepository, PaymentNotifier notifier) {
        this.paymentRepository = paymentRepository;
        this.notifier = notifier;
    }
}
```

```java
@Component
public class ConsolePaymentNotifier implements PaymentNotifier {

    @Override
    public void notifyCreated(Payment payment) {
        System.out.println("Pagamento criado: " + payment.getId());
    }
}
```

O service não sabe se a notificação vai para console, email, SMS ou fila RabbitMQ. Ele depende do contrato `PaymentNotifier`.

## Código completo

Este é o conjunto mínimo de classes que representa a versão melhorada do exemplo.

```java
package br.edu.upf.paymentapi.service;

import java.util.Map;

public record ProcessedPayment(Map<String, Object> metadata) {
}
```

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import java.math.BigDecimal;

public interface PaymentProcessor {

    String getType();

    ProcessedPayment process(CreatePaymentRequest request);

    BigDecimal calculateFee(BigDecimal amount);
}
```

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import java.math.BigDecimal;
import java.util.Map;
import org.springframework.stereotype.Service;

@Service
public class PixPaymentProcessor implements PaymentProcessor {

    @Override
    public String getType() {
        return "PIX";
    }

    @Override
    public ProcessedPayment process(CreatePaymentRequest request) {
        return new ProcessedPayment(Map.of(
                "pixKey", request.pixKey(),
                "qrCode", "pix-" + System.currentTimeMillis()
        ));
    }

    @Override
    public BigDecimal calculateFee(BigDecimal amount) {
        return BigDecimal.ZERO;
    }
}
```

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import java.math.BigDecimal;
import java.util.Map;
import org.springframework.stereotype.Service;

@Service
public class CreditCardPaymentProcessor implements PaymentProcessor {

    @Override
    public String getType() {
        return "CREDIT_CARD";
    }

    @Override
    public ProcessedPayment process(CreatePaymentRequest request) {
        return new ProcessedPayment(Map.of(
                "cardNumber", maskCard(request.cardNumber()),
                "processor", detectProcessor(request.cardNumber())
        ));
    }

    @Override
    public BigDecimal calculateFee(BigDecimal amount) {
        return amount.multiply(new BigDecimal("0.03"));
    }

    private String maskCard(String cardNumber) {
        return "************" + cardNumber.substring(cardNumber.length() - 4);
    }

    private String detectProcessor(String cardNumber) {
        if (cardNumber.startsWith("4")) {
            return "Visa";
        }
        if (cardNumber.startsWith("5")) {
            return "Mastercard";
        }
        return "Unknown";
    }
}
```

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import java.math.BigDecimal;
import java.util.Map;
import org.springframework.stereotype.Service;

@Service
public class BoletoPaymentProcessor implements PaymentProcessor {

    @Override
    public String getType() {
        return "BOLETO";
    }

    @Override
    public ProcessedPayment process(CreatePaymentRequest request) {
        return new ProcessedPayment(Map.of(
                "boletoNumber", "34191.79001 01043.510047 91020.150008 1",
                "dueDate", request.dueDate()
        ));
    }

    @Override
    public BigDecimal calculateFee(BigDecimal amount) {
        return new BigDecimal("2.50");
    }
}
```

```java
package br.edu.upf.paymentapi.service;

import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ResponseStatusException;

@Component
public class PaymentProcessorResolver {

    private final Map<String, PaymentProcessor> processors;

    public PaymentProcessorResolver(List<PaymentProcessor> processors) {
        this.processors = processors.stream()
                .collect(Collectors.toMap(PaymentProcessor::getType, Function.identity()));
    }

    public PaymentProcessor resolve(String type) {
        PaymentProcessor processor = processors.get(type);

        if (processor == null) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Tipo de pagamento nao suportado");
        }

        return processor;
    }
}
```

## Erros comuns

- Criar interface antes de existir mais de uma implementação real.
- Colocar regra de negócio no controller.
- Criar uma interface gigante chamada `PaymentServiceInterface` com todos os métodos do sistema.
- Usar SOLID para complicar uma aula simples.
- Confundir repository com regra de negócio. Repository acessa o MongoDB; regra de negócio fica no service.
- Usar `@Autowired` em atributo em vez de injeção por construtor.

## Resumo

SOLID não é sobre escrever mais código. É sobre organizar responsabilidades para que o sistema aguente mudança.

No exemplo da Payment API:

- SRP separa responsabilidades.
- OCP permite adicionar novo tipo de pagamento sem mexer no service principal.
- LSP garante que qualquer processador siga o mesmo contrato.
- ISP evita interfaces grandes demais.
- DIP reduz acoplamento usando abstrações e injeção de dependência.

Na próxima parte prática, vamos codar primeiro a versão legacy, ignorando SOLID de propósito. Depois vamos transformar esse código em uma versão mais limpa usando os princípios vistos aqui.
