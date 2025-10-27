# Princípios SOLID: Fundamentos da Programação Orientada a Objetos

## O que é SOLID?

SOLID é um acrônimo que representa **5 princípios fundamentais** da Programação Orientada a Objetos (POO). Estes princípios foram criados para ajudar desenvolvedores a escrever código mais limpo, manutenível e escalável.

### Origem dos Princípios SOLID

Os princípios SOLID foram introduzidos por **Robert C. Martin** (conhecido como "Uncle Bob") em 1995, no artigo "The Principles of OoD" (Object-Oriented Design). Em 2002, ele consolidou esses princípios no livro "Agile Software Development, Principles, Patterns, and Practices".

### Por que SOLID é Importante?

Imagine construir uma casa. Se você não seguir princípios básicos de arquitetura (fundação sólida, estrutura bem planejada, materiais adequados), a casa pode desabar ou ser muito difícil de reformar. O mesmo acontece com software!

**Código sem SOLID**:
- Difícil de manter e modificar
- Propenso a bugs
- Difícil de testar
- Acoplado demais (mudança em uma parte quebra outras)

**Código com SOLID**:
- Fácil de manter e expandir
- Menos bugs
- Fácil de testar
- Componentes independentes

## Os 5 Princípios SOLID

### 1. Single Responsibility Principle (SRP) - Princípio da Responsabilidade Única

> **"Uma classe deve ter apenas uma razão para mudar"**

#### Analogia do Mundo Real: Restaurante

Imagine um restaurante onde o mesmo funcionário faz **tudo**:
- Cozinha os pratos
- Atende os clientes
- Lava a louça
- Limpa as mesas
- Cobra as contas
- Gerencia o estoque

**Problema**: Se o funcionário ficar doente, o restaurante para! E se você quiser melhorar o atendimento, precisa mexer no mesmo código que cozinha.

**Solução SOLID**: Cada funcionário tem **UMA** responsabilidade:
- **Chef**: Só cozinha
- **Garçom**: Só atende
- **Lavador**: Só lava louça
- **Caixa**: Só cobra

#### Exemplo de Violação (Código Ruim)

```typescript
// VIOLA SRP: Esta classe faz MUITAS coisas!
export class LegacyPaymentService {
  async createPayment(createPaymentDto: CreateLegacyPaymentDto) {
    // Responsabilidade 1: Validação
    if (!createPaymentDto.customerName) {
      throw new Error('Nome do cliente é obrigatório');
    }

    // Responsabilidade 2: Processamento de pagamento
    let processedPayment;
    if (createPaymentDto.type === 'CREDIT_CARD') {
      processedPayment = await this.processCreditCard(createPaymentDto);
    } else if (createPaymentDto.type === 'PIX') {
      processedPayment = await this.processPix(createPaymentDto);
    }

    // Responsabilidade 3: Cálculo de taxa
    const fee = this.calculateFee(createPaymentDto.amount, createPaymentDto.type);

    // Responsabilidade 4: Aplicação de regras de negócio
    const status = this.determineStatus(createPaymentDto.amount, createPaymentDto.type);

    // Responsabilidade 5: Persistência no banco
    const payment = await this.prisma.payment.create({ data: formattedData });

    // Responsabilidade 6: Envio de email
    await this.sendEmail(payment);

    // Responsabilidade 7: Log
    console.log(`Pagamento ${payment.id} criado com sucesso`);

    return payment;
  }
}
```

**Problemas**:
- Se mudar validação, pode quebrar processamento
- Se mudar regras de negócio, pode afetar email
- Difícil de testar cada responsabilidade separadamente
- Difícil de reutilizar partes do código

#### Exemplo de Aplicação Correta (Código Bom)

```typescript
// APLICA SRP: Cada classe tem UMA responsabilidade

// Classe 1: Só processa PIX
export class PixService {
  async process(valor: number, metadados: Record<string, unknown>) {
    // Só processa PIX - nada mais!
    return {
      idTransacao: this.gerarIdTransacao(),
      status: 'PROCESSANDO',
      dadosAdicionais: { /* dados do PIX */ }
    };
  }
}

// Classe 2: Só calcula taxas
export class TaxaCalculator {
  calcularTaxa(valor: number, tipo: string): number {
    // Só calcula taxas - nada mais!
    if (tipo === 'PIX') return 0;
    if (tipo === 'CARTAO') return valor * 0.03;
    return 0;
  }
}

// Classe 3: Só orquestra pagamentos
export class ServicoPagamento {
  constructor(
    private pixService: PixService,
    private taxaCalculator: TaxaCalculator,
    private repositorio: PaymentRepository
  ) {}

  async criarPagamento(dados: CriarPagamentoDto) {
    // Só orquestra - delega responsabilidades específicas
    const pagamentoProcessado = await this.pixService.process(dados.valor, dados.metadados);
    const taxa = this.taxaCalculator.calcularTaxa(dados.valor, dados.tipo);
    return await this.repositorio.criar({ /* dados */ });
  }
}
```

**Benefícios**:
- Cada classe tem uma responsabilidade clara
- Fácil de testar cada parte separadamente
- Fácil de modificar uma responsabilidade sem afetar outras
- Código mais legível e organizado

---

### 2. Open/Closed Principle (OCP) - Princípio Aberto-Fechado

> **"Aberto para extensão, fechado para modificação"**

#### Analogia do Mundo Real: Receita de Bolo

Imagine uma **receita de bolo básica**:
- 2 xícaras de farinha
- 1 xícara de açúcar
- 3 ovos
- 1 xícara de leite

**Problema**: Se você quiser fazer um bolo de chocolate, precisa **modificar** a receita original.

**Solução SOLID**: A receita base **não muda**, mas você pode **adicionar** ingredientes:
- Receita base: sempre a mesma
- Bolo de chocolate: adiciona cacau
- Bolo de morango: adiciona morangos
- Bolo de coco: adiciona coco

#### Exemplo de Violação (Código Ruim)

```typescript
// VIOLA OCP: Preciso MODIFICAR código existente para adicionar novo tipo
export class LegacyPaymentService {
  async createPayment(createPaymentDto: CreateLegacyPaymentDto) {
    let processedPayment;
    
    // if/else gigante - preciso MODIFICAR para adicionar novo tipo
    if (createPaymentDto.type === 'CREDIT_CARD') {
      processedPayment = await this.processCreditCard(createPaymentDto);
    } else if (createPaymentDto.type === 'PIX') {
      processedPayment = await this.processPix(createPaymentDto);
    } else if (createPaymentDto.type === 'BOLETO') {
      processedPayment = await this.processBoleto(createPaymentDto);
    } else if (createPaymentDto.type === 'PAYPAL') { // MODIFIQUEI CÓDIGO EXISTENTE!
      processedPayment = await this.processPayPal(createPaymentDto);
    } else if (createPaymentDto.type === 'CRYPTO') { // MODIFIQUEI NOVAMENTE!
      processedPayment = await this.processCrypto(createPaymentDto);
    }
    
    return processedPayment;
  }
}
```

**Problemas**:
- Toda vez que adicionar novo tipo, preciso modificar código existente
- Risco de quebrar funcionalidades existentes
- Código fica cada vez maior e mais complexo
- Difícil de testar todas as combinações

#### Exemplo de Aplicação Correta (Código Bom)

```typescript
// APLICA OCP: Strategy Pattern - adiciono sem modificar código existente

// Interface que define o contrato
export interface IPaymentStrategy {
  process(valor: number, metadados: Record<string, unknown>): Promise<DadosProcessamentoPagamento>;
  calcularTaxa(valor?: number): number;
  obterTipo(): string;
}

// Implementação PIX
export class PixService implements IPaymentStrategy {
  async process(valor: number, metadados: Record<string, unknown>) {
    // Implementação específica do PIX
  }
  calcularTaxa() { return 0; }
  obterTipo() { return 'PIX'; }
}

// Implementação Cartão
export class CreditCardService implements IPaymentStrategy {
  async process(valor: number, metadados: Record<string, unknown>) {
    // Implementação específica do Cartão
  }
  calcularTaxa(valor: number) { return valor * 0.03; }
  obterTipo() { return 'CARTAO_CREDITO'; }
}

// Classe principal - NÃO PRECISA MUDAR para adicionar novos tipos
export class ServicoPagamento {
  private estrategias: Map<string, IPaymentStrategy>;

  constructor(
    private servicoPix: PixService,
    private servicoCartao: CreditCardService,
    // Para adicionar PayPal, só preciso injetar aqui
    // private servicoPayPal: PayPalService,
  ) {
    this.estrategias = new Map();
    this.estrategias.set('PIX', servicoPix);
    this.estrategias.set('CARTAO_CREDITO', servicoCartao);
    // this.estrategias.set('PAYPAL', servicoPayPal); // ADICIONO SEM MODIFICAR!
  }

  async criarPagamento(dadosPagamento: CriarPagamentoDto) {
    // Busca estratégia sem if/else - código não muda!
    const estrategia = this.estrategias.get(dadosPagamento.tipo);
    return await estrategia.process(dadosPagamento.valor, dadosPagamento.metadados);
  }
}
```

**Benefícios**:
- Adiciono novos tipos sem modificar código existente
- Código existente continua funcionando
- Fácil de testar cada estratégia separadamente
- Código mais limpo e organizado

---

### 3. Liskov Substitution Principle (LSP) - Princípio de Substituição de Liskov

> **"Objetos de uma superclasse devem ser substituíveis por objetos de subclasses"**

#### Analogia do Mundo Real: Funcionários e Cargos

Imagine uma empresa onde você tem **funcionários** que podem ocupar diferentes **cargos**:
- **Gerente**: Pode ser substituído por qualquer pessoa que "saiba gerenciar"
- **Vendedor**: Pode ser substituído por qualquer pessoa que "saiba vender"
- **Contador**: Pode ser substituído por qualquer pessoa que "saiba contabilizar"

**Problema**: Se você contratar um "vendedor" que na verdade não sabe vender (só sabe fazer relatórios), ele não pode substituir um vendedor real!

**Solução SOLID**: Todos os vendedores devem "implementar a interface de vendedor":
- Saber apresentar produtos
- Saber fechar vendas
- Saber atender clientes
- Saber fazer follow-up

Qualquer pessoa que implemente essa interface pode substituir qualquer outro vendedor.

#### Exemplo de Violação (Código Ruim)

```typescript
// VIOLA LSP: Preciso MODIFICAR código existente para adicionar novo tipo
export class LegacyPaymentService {
  async createPayment(createPaymentDto: CreateLegacyPaymentDto) {
    // if/else gigante - preciso MODIFICAR para adicionar novo tipo
    let processedPayment;
    if (createPaymentDto.type === 'CREDIT_CARD') {
      processedPayment = await this.processCreditCard(createPaymentDto);
    } else if (createPaymentDto.type === 'PIX') {
      processedPayment = await this.processPix(createPaymentDto);
    } else if (createPaymentDto.type === 'BOLETO') {
      processedPayment = await this.processBoleto(createPaymentDto);
    } else if (createPaymentDto.type === 'PAYPAL') { // MODIFIQUEI CÓDIGO EXISTENTE!
      processedPayment = await this.processPayPal(createPaymentDto);
    } else if (createPaymentDto.type === 'CRYPTO') { // MODIFIQUEI NOVAMENTE!
      processedPayment = await this.processCrypto(createPaymentDto);
    }
    
    return processedPayment;
  }
}
```

**Problemas**:
- Toda vez que adicionar novo tipo, preciso modificar código existente
- Risco de quebrar funcionalidades existentes
- Código fica cada vez maior e mais complexo
- Difícil de testar todas as combinações

#### Exemplo de Aplicação Correta (Código Bom)

```typescript
// APLICA LSP: Strategy Pattern - adiciono sem modificar código existente

// Interface que define o contrato
export interface IPaymentStrategy {
  process(valor: number, metadados: Record<string, unknown>): Promise<DadosProcessamentoPagamento>;
  calcularTaxa(valor?: number): number;
  obterTipo(): string;
}

// Implementação PIX
export class PixService implements IPaymentStrategy {
  async process(valor: number, metadados: Record<string, unknown>) {
    // Implementação específica do PIX
  }
  calcularTaxa() { return 0; }
  obterTipo() { return 'PIX'; }
}

// Implementação Cartão
export class CreditCardService implements IPaymentStrategy {
  async process(valor: number, metadados: Record<string, unknown>) {
    // Implementação específica do Cartão
  }
  calcularTaxa(valor: number) { return valor * 0.03; }
  obterTipo() { return 'CARTAO_CREDITO'; }
}

// Classe principal - NÃO PRECISA MUDAR para adicionar novos tipos
export class ServicoPagamento {
  private estrategias: Map<string, IPaymentStrategy>;

  constructor(
    private servicoPix: PixService,
    private servicoCartao: CreditCardService,
    // Para adicionar PayPal, só preciso injetar aqui
    // private servicoPayPal: PayPalService,
  ) {
    this.estrategias = new Map();
    this.estrategias.set('PIX', servicoPix);
    this.estrategias.set('CARTAO_CREDITO', servicoCartao);
    // this.estrategias.set('PAYPAL', servicoPayPal); // ADICIONO SEM MODIFICAR!
  }

  async criarPagamento(dadosPagamento: CriarPagamentoDto) {
    // Busca estratégia sem if/else - código não muda!
    const estrategia = this.estrategias.get(dadosPagamento.tipo);
    return await estrategia.process(dadosPagamento.valor, dadosPagamento.metadados);
  }
}
```

**Benefícios**:
- Adiciono novos tipos sem modificar código existente
- Código existente continua funcionando
- Fácil de testar cada estratégia separadamente
- Código mais limpo e organizado

---

### 4. Interface Segregation Principle (ISP) - Princípio de Segregação de Interface

> **"Clientes não devem depender de interfaces que não usam"**

#### Analogia do Mundo Real: Controle Remoto

Imagine um **controle remoto gigante** com 100 botões:
- Botão de ligar TV
- Botão de volume
- Botão de canal
- Botão de lavar roupa
- Botão de ligar ar condicionado
- Botão de abrir portão
- Botão de ligar fogão
- ... (94 botões a mais)

**Problema**: Você só quer assistir TV, mas precisa carregar um controle gigante com 100 botões!

**Solução SOLID**: **Controles específicos**:
- **Controle da TV**: só botões de TV
- **Controle do ar**: só botões de ar condicionado
- **Controle do portão**: só botão de abrir/fechar

#### Exemplo de Violação (Código Ruim)

```typescript
// VIOLA ISP: Interface gigante com muitos métodos
export interface IPaymentService {
  // Métodos de pagamento
  processPayment(valor: number, tipo: string): Promise<any>;
  calculateFee(valor: number, tipo: string): number;
  
  // Métodos de email
  sendEmail(to: string, subject: string, body: string): Promise<void>;
  sendSMS(phone: string, message: string): Promise<void>;
  
  // Métodos de banco de dados
  savePayment(payment: any): Promise<any>;
  findPayment(id: string): Promise<any>;
  updatePayment(id: string, data: any): Promise<any>;
  deletePayment(id: string): Promise<void>;
  
  // Métodos de validação
  validateEmail(email: string): boolean;
  validatePhone(phone: string): boolean;
  validateCreditCard(card: string): boolean;
  
  // Métodos de relatório
  generateReport(startDate: Date, endDate: Date): Promise<any>;
  exportToExcel(data: any[]): Promise<Buffer>;
  exportToPDF(data: any[]): Promise<Buffer>;
}

// Cliente que só quer processar pagamentos
export class PaymentProcessor {
  constructor(private paymentService: IPaymentService) {}
  
  async process(valor: number, tipo: string) {
    // Forçado a depender de métodos que não usa
    const payment = await this.paymentService.processPayment(valor, tipo);
    const fee = this.paymentService.calculateFee(valor, tipo);
    
    // Mas não usa: sendEmail, sendSMS, savePayment, etc.
    return { payment, fee };
  }
}
```

**Problemas**:
- Cliente depende de métodos que não usa
- Interface muito grande e complexa
- Difícil de implementar (precisa implementar todos os métodos)
- Acoplamento desnecessário

#### Exemplo de Aplicação Correta (Código Bom)

```typescript
// APLICA ISP: Interfaces específicas e focadas

// Interface específica para processamento de pagamento
export interface IPaymentProcessor {
  process(valor: number, metadados: Record<string, unknown>): Promise<DadosProcessamentoPagamento>;
  calcularTaxa(valor?: number): number;
  obterTipo(): string;
}

// Interface específica para persistência
export interface IPaymentRepository {
  criar(dados: Omit<DadosPagamento, 'id' | 'dataCriacao' | 'dataAtualizacao'>): Promise<DadosPagamento>;
  buscarPorId(id: string): Promise<DadosPagamento | null>;
  buscarTodos(): Promise<DadosPagamento[]>;
  obterEstatisticas(): Promise<EstatisticasPagamento>;
}

// Interface específica para notificações
export interface INotificationService {
  enviarEmail(destinatario: string, assunto: string, corpo: string): Promise<void>;
  enviarSMS(telefone: string, mensagem: string): Promise<void>;
}

// Interface específica para validação
export interface IValidationService {
  validarEmail(email: string): boolean;
  validarTelefone(telefone: string): boolean;
  validarCartaoCredito(cartao: string): boolean;
}

// Cliente que só quer processar pagamentos
export class PaymentProcessor {
  constructor(
    private paymentService: IPaymentProcessor, // Só depende do que usa
    private repository: IPaymentRepository     // Só depende do que usa
  ) {}
  
  async process(valor: number, tipo: string, metadados: Record<string, unknown>) {
    // Usa apenas métodos necessários
    const payment = await this.paymentService.process(valor, metadados);
    const taxa = this.paymentService.calcularTaxa(valor);
    
    // Salva no banco
    const pagamentoSalvo = await this.repository.criar({
      tipo,
      valor: valor + taxa,
      status: 'PROCESSANDO',
      nomeCliente: metadados.nomeCliente,
      emailCliente: metadados.emailCliente,
      metadados: payment.dadosAdicionais
    });
    
    return pagamentoSalvo;
  }
}

// Cliente que só quer enviar notificações
export class NotificationManager {
  constructor(
    private notificationService: INotificationService, // Só depende do que usa
    private validationService: IValidationService       // Só depende do que usa
  ) {}
  
  async enviarNotificacaoPagamento(email: string, telefone: string, dados: any) {
    // Usa apenas métodos necessários
    if (this.validationService.validarEmail(email)) {
      await this.notificationService.enviarEmail(email, 'Pagamento Processado', 'Seu pagamento foi processado');
    }
    
    if (this.validationService.validarTelefone(telefone)) {
      await this.notificationService.enviarSMS(telefone, 'Pagamento processado com sucesso!');
    }
  }
}
```

**Benefícios**:
- Cada cliente depende apenas do que realmente usa
- Interfaces menores e mais focadas
- Fácil de implementar (só implementa o necessário)
- Menor acoplamento entre componentes

---

### 5. Dependency Inversion Principle (DIP) - Princípio da Inversão de Dependência

> **"Dependa de abstrações, não de implementações concretas"**

#### Analogia do Mundo Real: Tomada Elétrica

Imagine uma **tomada elétrica** na sua casa:
- Você **não precisa saber** como a energia é gerada
- Pode ser hidrelétrica, solar, eólica, nuclear...
- A **tomada é a abstração** - sempre funciona igual
- A **usina é a implementação** - pode mudar sem afetar a tomada

**Regra**: Dependa da **tomada** (abstração), não da **usina** (implementação).

#### Exemplo de Violação (Código Ruim)

```typescript
// VIOLA DIP: Dependência direta de implementações concretas

// Implementação concreta do banco
export class MySQLDatabase {
  async save(data: any) {
    // Código específico do MySQL
    console.log('Salvando no MySQL...');
  }
}

// Implementação concreta do email
export class GmailService {
  async sendEmail(to: string, subject: string, body: string) {
    // Código específico do Gmail
    console.log('Enviando email via Gmail...');
  }
}

// Cliente com dependências diretas
export class LegacyPaymentService {
  private database: MySQLDatabase;        // Dependência direta
  private emailService: GmailService;    // Dependência direta

  constructor() {
    // Cria instâncias diretamente
    this.database = new MySQLDatabase();
    this.emailService = new GmailService();
  }

  async createPayment(data: any) {
    // Processa pagamento
    const payment = { id: '123', amount: 100 };
    
    // Depende de implementação específica
    await this.database.save(payment);
    await this.emailService.sendEmail('user@email.com', 'Pagamento', 'Processado');
    
    return payment;
  }
}
```

**Problemas**:
- Se mudar de MySQL para PostgreSQL, precisa modificar código
- Se mudar de Gmail para Outlook, precisa modificar código
- Difícil de testar (não consegue mockar dependências)
- Código acoplado demais

#### Exemplo de Aplicação Correta (Código Bom)

```typescript
// APLICA DIP: Depende de abstrações, não implementações

// Abstração para banco de dados
export interface IDatabase {
  save(data: any): Promise<void>;
  findById(id: string): Promise<any>;
  findAll(): Promise<any[]>;
}

// Abstração para serviço de email
export interface IEmailService {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}

// Implementação concreta do MySQL
export class MySQLDatabase implements IDatabase {
  async save(data: any) {
    console.log('Salvando no MySQL...');
  }
  
  async findById(id: string) {
    console.log('Buscando no MySQL...');
  }
  
  async findAll() {
    console.log('Listando do MySQL...');
  }
}

// Implementação concreta do PostgreSQL
export class PostgreSQLDatabase implements IDatabase {
  async save(data: any) {
    console.log('Salvando no PostgreSQL...');
  }
  
  async findById(id: string) {
    console.log('Buscando no PostgreSQL...');
  }
  
  async findAll() {
    console.log('Listando do PostgreSQL...');
  }
}

// Implementação concreta do Gmail
export class GmailService implements IEmailService {
  async sendEmail(to: string, subject: string, body: string) {
    console.log('Enviando email via Gmail...');
  }
}

// Implementação concreta do Outlook
export class OutlookService implements IEmailService {
  async sendEmail(to: string, subject: string, body: string) {
    console.log('Enviando email via Outlook...');
  }
}

// Cliente que depende de abstrações
export class ServicoPagamento {
  constructor(
    private database: IDatabase,        // Depende de abstração
    private emailService: IEmailService // Depende de abstração
  ) {}

  async criarPagamento(data: any) {
    // Processa pagamento
    const payment = { id: '123', amount: 100 };
    
    // Usa abstrações - funciona com qualquer implementação
    await this.database.save(payment);
    await this.emailService.sendEmail('user@email.com', 'Pagamento', 'Processado');
    
    return payment;
  }
}

// Configuração com injeção de dependências
export class AppModule {
  static create() {
    // Pode trocar implementações facilmente
    const database = new MySQLDatabase();        // ou new PostgreSQLDatabase()
    const emailService = new GmailService();     // ou new OutlookService()
    
    return new ServicoPagamento(database, emailService);
  }
}
```

**Benefícios**:
- Posso trocar MySQL por PostgreSQL sem modificar código
- Posso trocar Gmail por Outlook sem modificar código
- Fácil de testar (posso usar mocks)
- Código mais flexível e desacoplado

---

## Clean Code e Boas Práticas

### O que é Clean Code?

**Clean Code** é código que é:
- **Fácil de entender** por outros desenvolvedores
- **Fácil de modificar** e estender
- **Fácil de testar** e debugar
- **Fácil de manter** ao longo do tempo

### Vantagens do Clean Code

1. **Manutenibilidade**: Fácil de corrigir bugs e adicionar funcionalidades
2. **Legibilidade**: Outros desenvolvedores entendem rapidamente
3. **Testabilidade**: Fácil de criar testes automatizados
4. **Reutilização**: Código pode ser reutilizado em outros projetos
5. **Colaboração**: Equipe trabalha melhor com código limpo

### Como Aplicar SOLID na Prática

#### 1. Identifique Responsabilidades
- Cada classe deve ter **uma** responsabilidade clara
- Se uma classe faz muitas coisas, divida em classes menores

#### 2. Use Abstrações
- Crie interfaces para definir contratos
- Dependa de interfaces, não de implementações concretas

#### 3. Aplique Strategy Pattern
- Use quando tiver múltiplas formas de fazer a mesma coisa
- Facilita adicionar novas estratégias sem modificar código existente

#### 4. Injeção de Dependências
- Injete dependências via construtor
- Facilita testes e troca de implementações

#### 5. Interfaces Pequenas e Focadas
- Crie interfaces específicas para cada necessidade
- Evite interfaces gigantes com muitos métodos

## Comparação: Código com SOLID vs sem SOLID

### Módulo Legacy-Payment (Sem SOLID)

```typescript
// Código ruim - viola todos os princípios SOLID
export class LegacyPaymentService {
  async createPayment(createPaymentDto: CreateLegacyPaymentDto) {
    // SRP: Faz validação, processamento, cálculo, persistência, email, log
    
    // Validação
    if (!createPaymentDto.customerName) {
      throw new Error('Nome do cliente é obrigatório');
    }
    
    // Processamento com if/else gigante
    let processedPayment;
    if (createPaymentDto.type === 'CREDIT_CARD') {
      processedPayment = await this.processCreditCard(createPaymentDto);
    } else if (createPaymentDto.type === 'PIX') {
      processedPayment = await this.processPix(createPaymentDto);
    }
    
    // Cálculo de taxa
    const fee = this.calculateFee(createPaymentDto.amount, createPaymentDto.type);
    
    // Persistência
    const payment = await this.prisma.payment.create({ data: formattedData });
    
    // Email
    await this.sendEmail(payment);
    
    // Log
    console.log(`Pagamento ${payment.id} criado com sucesso`);
    
    return payment;
  }
}
```

**Problemas**:
- Difícil de manter e modificar
- Difícil de testar
- Acoplado demais
- Viola todos os princípios SOLID

### Módulo Payment (Com SOLID)

```typescript
// Código bom - aplica todos os princípios SOLID

// SRP: Cada classe tem uma responsabilidade
export class PixService implements IPaymentStrategy {
  async process(valor: number, metadados: Record<string, unknown>) {
    // Só processa PIX
  }
}

export class ServicoPagamento {
  constructor(
    @Inject('IPaymentRepository')
    private repositorioPagamento: IPaymentRepository, // DIP: Depende de abstração
    private servicoPix: PixService,                   // DIP: Injeção de dependência
  ) {
    // OCP: Estratégias podem ser adicionadas sem modificar código
    this.estrategias = new Map();
    this.estrategias.set('PIX', servicoPix);
  }

  async criarPagamento(dadosPagamento: CriarPagamentoDto) {
    // OCP: Busca estratégia sem if/else
    const estrategia = this.estrategias.get(dadosPagamento.tipo);
    
    // LSP: Qualquer estratégia pode ser usada
    const pagamentoProcessado = await estrategia.process(/* ... */);
    
    // SRP: Delega responsabilidades específicas
    const taxa = estrategia.calcularTaxa(dadosPagamento.valor);
    const pagamento = await this.repositorioPagamento.criar(/* ... */);
    
    return pagamento;
  }
}
```

**Benefícios**:
- Fácil de manter e modificar
- Fácil de testar
- Desacoplado
- Aplica todos os princípios SOLID

## Conclusão

Os princípios SOLID são fundamentais para escrever código de qualidade. Eles nos ajudam a criar software que é:

- **Manutenível**: Fácil de modificar e estender
- **Testável**: Fácil de criar testes automatizados
- **Reutilizável**: Código pode ser usado em diferentes contextos
- **Legível**: Fácil de entender por outros desenvolvedores
- **Flexível**: Fácil de adaptar a mudanças

---

**Fonte**: Material baseado no curso ["SOLID com TypeScript" da Alura](https://www.alura.com.br/curso-online-solid-typescript).

