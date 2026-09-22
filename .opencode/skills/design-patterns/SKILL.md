---
name: design-patterns
description: Padrões de projeto e boas práticas de código estabelecidas para o backend Java 25 + Spring + PostgreSQL + Flyway + JUnit/Mockito, complementando a arquitetura hexagonal. Use ao implementar casos de uso, ao revisar código, ou ao decidir como resolver algo não coberto explicitamente pela especificação.
---

# Design Patterns e Boas Práticas — Java 25 / Spring

## Quando usar
Use ao implementar casos de uso, ao revisar código, ou ao decidir como resolver algo não coberto explicitamente pela especificação técnica. Complementa a skill `hexagonal-architecture` (que define *onde* cada coisa mora) descrevendo *como* implementar bem dentro de cada camada; destina-se ao time de desenvolvimento backend Java 25 + Spring + PostgreSQL.

O conteúdo está organizado em duas partes: **Parte A — Boas práticas estabelecidas** (base fixa, validada pela comunidade: Effective Java, GoF, Clean Code, práticas idiomáticas de Spring) e **Parte B — Padrões do time** (seção reservada para convenções específicas do time). Sempre que houver conflito, **Parte B prevalece sobre a Parte A**.

## Instruções (passo a passo)
Ao implementar ou revisar código, aplique as regras na ordem em que aparecem: comece pelos princípios gerais (seção 1), consulte os padrões por categoria — criacionais (seção 2), estruturais (seção 3), comportamentais (seção 4) —, aplique as regras de tratamento de exceções (seção 5), validação (seção 6), injeção de dependência (seção 7), persistência (seção 8) e transações (seção 9), garanta os padrões de teste (seção 10) e finalize com o checklist de code smells (seção 13) para detectar violações. Se um comportamento necessário não estiver coberto, proponha-o e registre-o na Parte B conforme o template da seção final.

# PARTE A — Boas práticas estabelecidas

## 1. Princípios gerais

- **SOLID** como guia geral, com ênfase em:
  - *SRP*: uma classe/caso de uso resolve uma única responsabilidade de negócio.
  - *DIP*: dependências apontam para abstrações (`port`), nunca para implementações concretas — já formalizado na skill `hexagonal-architecture`.
- **Composição antes de herança**: preferir injetar comportamento (Strategy, delegação) a criar hierarquias de herança para variação de regra de negócio.
- **Imutabilidade por padrão**: usar `record` (Java 16+, disponível e preferido em Java 25) para Value Objects, DTOs de aplicação, Commands/Queries e respostas. Classes mutáveis só quando houver razão explícita (ex.: builders intermediários).
- **Fail-fast & Guard Clauses**: validar pré-condições e invariantes logo no início do método/construtor e lançar exceção/retornar imediatamente. Evita aninhamento desnecessário de blocos `if/else`.
- **KISS (Keep It Simple, Stupid)**: evitar abstrações prematuras ou soluções complexas quando uma abordagem simples resolve o problema.
- **Código Autoexplicativo (Self-Documenting Code)**: nomes claros de variáveis e métodos dispensam comentários óbvios no código.

### 1.1 Extração de Código e Princípio DRY (Don't Repeat Yourself)
- **Constantes de Exceção e Erro**: mensagens de erro, textos de exceção, mensagens de validação e códigos de erro repetidos mais de uma vez devem ser extraídos obrigatoriamente para constantes `private static final` na classe ou mantidos em enums/classes de mensagens centralizadas do módulo. Proibido usar "Strings Mágicas" espalhadas em `throw new ...`.
- **Extração de Métodos de Validação**: blocos de validação repetidos ou checagens condicionais com mais de duas expressões lógicas devem ser isolados em métodos auxiliares privados e nomeados de forma declarativa (ex.: `validarElegibilidadeCliente(...)` ou `isClienteElegivel(...)`).

### 1.2 Formatação, Densidade Vertical e Quebra de Linhas
- **Limite Horizontal**: o limite da largura de linha do projeto é de **120 caracteres**.
- **Densidade Vertical**: mantenha instruções relacionadas agrupadas sem linhas em branco intercaladas. Use linha em branco apenas para separar etapas lógicas distintas dentro de um método (ex.: separação das fases *Arrange*, *Act*, *Assert*).
- **Chamadas de Métodos e Argumentos**:
  - Chamadas de método simples, instanciações e repasse de parâmetros devem permanecer na **mesma linha**, a menos que ultrapassem o limite de 120 caracteres.
  - Não quebre linhas entre parâmetros de métodos pequenos (ex.: `metodo(a, b, c)` deve ficar em 1 linha).
- **Encadeamento de Métodos (Fluent APIs / Streams / Builders / Mockito)**:
  - Encadeamentos curtos que couberem em até 120 caracteres devem permanecer em **linha única** (ex.: `var lista = itens.stream().filter(Item::isAtivo).toList();`).
  - Quando a instrução encadeada exceder 120 caracteres, quebre cada chamada na linha seguinte, mantendo o operador de navegação ponto (`.`) sempre no **início da nova linha** com a devida indentação.

#### Exemplo de Formatação e Extração (Java 25)

```java
// ❌ RUIM: Mensagens de erro duplicadas/literais, quebras de linha arbitrárias e aninhamento
public class ProcessarPagamentoUseCase {
    public void executar(PagamentoCommand command) {
        if (command.valor() == null || command.valor().compareTo(BigDecimal.ZERO) <= 0) {
            throw new PagamentoInvalidoException("O valor do pagamento deve ser maior que zero");
        }
        var resultado = clienteRepository
            .buscarPorId(command.clienteId())
            .orElseThrow(() -> new ClienteNaoEncontradoException("Cliente não encontrado"));
        
        // ...
    }
}

// ✅ BOM: Constantes extraídas, guard clauses, fluidez e formatação limpa em linha única quando curta
public class ProcessarPagamentoUseCase {
    private static final String MSG_VALOR_INVALIDO = "O valor do pagamento deve ser maior que zero";
    private static final String MSG_CLIENTE_NAO_ENCONTRADO = "Cliente não encontrado";

    public void executar(PagamentoCommand command) {
        validarValor(command.valor());

        var cliente = clienteRepository.buscarPorId(command.clienteId())
            .orElseThrow(() -> new ClienteNaoEncontradoException(MSG_CLIENTE_NAO_ENCONTRADO));

        // ...
    }

    private void validarValor(BigDecimal valor) {
        if (valor == null || valor.compareTo(BigDecimal.ZERO) <= 0) {
            throw new PagamentoInvalidoException(MSG_VALOR_INVALIDO);
        }
    }
}
```

## 2. Padrões criacionais

### 2.1 Static Factory Method / Builder

- Para entidades de domínio com múltiplos campos ou regras de criação, preferir um **factory method estático nomeado** (`Pedido.criar(...)`) a um construtor público genérico — o nome comunica a intenção e é o lugar natural para validar invariantes.
- Para objetos com muitos campos opcionais (ex.: filtros de busca, requests complexos), usar **Builder** (manual ou via `record` com métodos `with*` que retornam nova instância).

```java
public record Pedido(PedidoId id, ClienteId clienteId, List<ItemPedido> itens, StatusPedido status) {

    public static Pedido criar(ClienteId clienteId, List<ItemPedido> itens) {
        if (itens == null || itens.isEmpty()) {
            throw new PedidoInvalidoException("Pedido deve ter ao menos um item");
        }
        return new Pedido(PedidoId.novo(), clienteId, List.copyOf(itens), StatusPedido.CRIADO);
    }
}
```

### 2.2 Factory de domínio para regras de criação complexas

Quando a criação de um objeto de domínio depende de múltiplas outras entidades ou de uma regra de negócio não trivial, extrair um `domain.service` do tipo *Factory* (ex.: `PedidoFactory`) em vez de sobrecarregar o Use Case com lógica de montagem.

---

## 3. Padrões estruturais

### 3.1 Adapter

Já formalizado pela arquitetura hexagonal — todo Output Port tem exatamente um Adapter que traduz entre o mundo externo (JPA, HTTP client) e o modelo de domínio. Regra adicional: **um Adapter nunca aplica regra de negócio**, apenas traduz e delega — se houver decisão de negócio dentro de um Adapter, é code smell.

### 3.2 Facade (opcional, quando útil)

Quando um Use Case precisa orquestrar múltiplos Output Ports/serviços de domínio em uma sequência não trivial, isolar essa orquestração em um método claro do próprio Use Case (ele já é, por definição, uma fachada da funcionalidade). Não criar uma camada de Facade adicional só para "esconder" o Use Case — isso duplica a responsabilidade que o Input Port já tem.

### 3.3 DTO Mapping explícito

- Cada tradução entre camadas (`Entity JPA ↔ domain.model`, `request HTTP ↔ Command`, `domain.model ↔ response HTTP`) deve ter um **Mapper dedicado** (classe `*Mapper` ou método estático `to*`/`from*`), nunca lógica de mapeamento espalhada dentro do Controller/Adapter.
- Ferramenta padrão: **MapStruct** para mapeamentos mecânicos (reduz boilerplate e erra menos que mapeamento manual); mapeamento manual é aceitável para casos simples (1–3 campos) ou quando há lógica de tradução não trivial.
- Mapper nunca lança exceção de negócio — apenas traduz estrutura.

---

## 4. Padrões comportamentais

### 4.1 Strategy

Usar quando existir **variação de algoritmo/regra selecionada em tempo de execução** (ex.: diferentes formas de cálculo de frete, diferentes validadores por tipo de documento). Implementar como interface + implementações injetadas via Spring (`Map<TipoX, EstrategiaX>` resolvido automaticamente pelo Spring a partir de múltiplos beans, ou um `enum` com lógica associada quando as opções são fixas e conhecidas em tempo de compilação).

```java
public interface CalculadoraFrete {
    boolean suporta(TipoEntrega tipo);
    Dinheiro calcular(Pedido pedido);
}
```

### 4.2 Template Method

Usar quando múltiplos casos de uso compartilham o mesmo esqueleto de passos, mas variam um passo específico (ex.: "validar → processar → notificar", variando apenas "processar"). Preferir isso a duplicar o esqueleto em cada Use Case.

### 4.3 Specification (regras de negócio compostas)

Para regras de negócio que se combinam (ex.: elegibilidade de desconto, filtros de busca complexos), preferir o padrão **Specification** a `if/else` encadeados extensos — cada regra vira uma unidade nomeada, testável isoladamente e combinável (`and`/`or`/`not`).

### 4.4 Domain Events (quando aplicável)

Quando uma ação de negócio dispara efeitos colaterais em outros contextos (ex.: "pedido criado" dispara notificação e atualização de estoque), preferir publicar um **evento de domínio** (`PedidoCriadoEvent`) via `ApplicationEventPublisher` do Spring a acoplar o Use Case diretamente a múltiplos Output Ports não relacionados à sua responsabilidade principal. Os listeners residem em `adapters` (nunca a lógica de reação em `domain`).

### 4.5 Chain of Responsibility

Útil para pipelines de validação/transformação em série (ex.: validações de request antes de virar Command). Usar quando a ordem e composição das etapas podem variar; para uma lista fixa e pequena de validações, Bean Validation (`@Valid`) já resolve sem necessidade do padrão.

---

## 5. Tratamento de exceções

- **Hierarquia de exceções de domínio**: base comum (ex.: `DomainException` abstrata ou `sealed`, aproveitando `sealed classes` do Java 25 quando o conjunto de exceções de um contexto é fechado e conhecido), com subtipos específicos por regra violada.
- **Tradução na borda**: exceções de domínio são traduzidas para respostas HTTP **apenas** em `adapters.in.web`, via `@RestControllerAdvice` centralizado. Nenhum Controller individual deve conter `try/catch` para tradução manual de erro — isso pertence ao *exception handler* global.
- **Formato de erro padronizado**: preferir `application/problem+json` (RFC 7807) como contrato de erro da API, com campos consistentes (`type`, `title`, `status`, `detail`, `instance` + extensões do domínio quando necessário).
- Exceções técnicas de infraestrutura (`DataAccessException`, `SQLException`, etc.) nunca vazam para fora de `adapters.out` — são capturadas e traduzidas para uma exceção de aplicação/domínio significativa antes de subir.

---

## 6. Validação

- **Validação estrutural/sintática** (campo obrigatório, formato, tamanho) na borda de entrada, com Bean Validation (`@NotNull`, `@Size`, etc.) nos DTOs de `adapters.in.web`.
- **Validação de invariante/regra de negócio** dentro do próprio objeto de domínio (construtor/factory method) ou em Domain Service — nunca duplicada no Controller.
- Não usar Bean Validation em objetos de domínio (`domain.model`) — isso violaria a regra de dependência (anotações `jakarta.validation` acopladas ao domínio são toleráveis por serem uma especificação, mas o padrão do projeto é manter o domínio livre até de anotações de validação, favorecendo validação programática nos factory methods para manter consistência com "domínio sem anotações externas").

---

## 7. Injeção de dependência

- **Injeção via construtor sempre** — nunca `@Autowired` em campo. Facilita testes unitários (construção direta com mocks) e deixa dependências explícitas.
- Classes com muitas dependências injetadas (ex.: mais de 5–6) são sinal de violação de SRP — considerar quebrar o Use Case.
- Beans de Use Case (`@Service`) e Adapter (`@Component`) são *stateless* — estado de execução nunca fica em campo de instância compartilhado.

---

## 8. Persistência (Repository)

- Interface do Output Port (`XyzRepositoryPort`) expressa operações em linguagem de negócio (`salvar`, `buscarPorId`, `buscarPedidosPendentes`), nunca em termos de query técnica.
- Consultas complexas/dinâmicas: usar **Specification do Spring Data JPA** ou Criteria API dentro do Adapter — nunca vazar `Specification`/`Criteria` para a camada de aplicação.
- Paginação: Output Port expõe tipos próprios de domínio/aplicação (`PaginaResultado<T>`), não `org.springframework.data.domain.Page` diretamente — evita acoplar a assinatura do Port a uma classe do Spring Data.

---

## 9. Transações

- `@Transactional` delimita a fronteira no **Use Case** (camada de aplicação), cobrindo exatamente as operações que precisam ser atômicas.
- Regra: uma transação não deve atravessar chamada a sistemas externos (HTTP client, fila) — apenas operações de persistência. Efeitos colaterais externos disparam após o commit (ex.: `@TransactionalEventListener(phase = AFTER_COMMIT)`).

---

## 10. Testes unitários (JUnit 5 + Mockito)

- **Padrão AAA** (Arrange / Act / Assert) em todo teste, com comentários ou espaçamento em branco separando as três seções.
- **Nomenclatura de teste** descritiva do comportamento: `deveLancarExcecao_quandoPedidoSemItens()` ou (BDD-style) `criarPedido_semItens_deveLancarPedidoInvalidoException()`.
- **Test Data Builder / Object Mother** para construir objetos de domínio/DTOs de teste complexos, evitando duplicação de setup em cada teste.
- **Mockito — regras**:
  - Mockar apenas os **Output Ports** (interfaces) nos testes de Use Case; nunca mockar classes de domínio (`Pedido`, Value Objects) — instanciá-las de verdade.
  - Preferir `verify` para validar interação relevante (ex.: "o pedido foi salvo"), evitando *over-verification* de chamadas irrelevantes ao comportamento testado.
  - Evitar `@Mock` para tudo por padrão — se um Use Case precisa mockar muitas dependências para ser testado, é sinal de SRP violado (ver seção 7).
  - Não usar `Mockito.mock()` para classes de domínio simples/Value Objects — isso indica que a classe deveria ser instanciada diretamente.
- Testes de domínio puro (`domain/**`) não usam Mockito nem Spring — são testes de unidade clássicos, apenas `assertEquals`/`assertThrows`.
- Cobertura mínima esperada em `domain` e `application.usecase`: todos os cenários de regra de negócio descritos na especificação (caminho feliz, bordas, exceções) — cobertura numérica é consequência, não o objetivo em si.

---

## 11. Null handling

- Métodos que podem legitimamente não retornar valor usam `Optional<T>` como tipo de retorno (nunca como parâmetro de método ou campo de classe).
- Evitar retornar `null` em Output Ports — usar `Optional` (`buscarPorId` retorna `Optional<Pedido>`) ou lançar exceção de domínio quando ausência é uma condição de erro esperada pelo caso de uso.

---

## 12. Java 25 — recursos idiomáticos a preferir

- **`record`** para Value Objects, DTOs, Commands/Queries e respostas imutáveis.
- **`sealed interface`/`sealed class`** para hierarquias fechadas e conhecidas (ex.: tipos de exceção de domínio de um contexto, estados de uma máquina de estados finita), aproveitando `switch` com *pattern matching* exaustivo no lugar de `if/else` encadeado.
- **Pattern matching for switch** para lidar com hierarquias `sealed` de forma exaustiva e sem `default` "escondendo" caso não tratado.
- **Virtual Threads**: usar como padrão para chamadas de I/O bloqueante em contextos de alta concorrência quando o time optar por isso explicitamente — não é obrigatório por padrão, deve ser decisão registrada na Parte B.

---

## 13. Checklist de code smells (para uso na Fase 3 — code review)

| # | Smell | Como identificar | Por que é um problema |
|---|---|---|---|
| S1 | Use Case com mais de 5-6 dependências injetadas | Contagem de parâmetros do construtor | Indica violação de SRP, dificulta teste e manutenção |
| S2 | Lógica de negócio dentro de Controller ou Adapter | Condicionais/decisões de negócio fora de `domain`/`application` | Regra de negócio não testável isoladamente, duplicável |
| S3 | Mapeamento manual espalhado (sem Mapper dedicado) | Conversão de objeto feita inline em múltiplos lugares | Duplicação, risco de inconsistência entre pontos de mapeamento |
| S4 | `try/catch` de tradução de erro dentro de Controller individual | Bloco `catch` retornando `ResponseEntity` de erro manualmente | Deveria estar centralizado no `@RestControllerAdvice` |
| S5 | Mock de classe de domínio simples | `Mockito.mock(Pedido.class)` ou similar | Classe deveria ser instanciada de verdade; teste perde valor |
| S6 | `@Transactional` em Controller ou Adapter | Anotação fora de `application.usecase` | Fronteira transacional mal definida, risco de transação "vazando" para chamadas externas |
| S7 | DTO HTTP reaproveitado como Command/domínio | Mesmo objeto atravessando `adapters.in` até `domain` | Acopla contrato externo ao modelo interno |
| S8 | Teste sem `Arrange/Act/Assert` claro, múltiplas asserções não relacionadas | Teste longo cobrindo vários comportamentos distintos | Difícil identificar causa de falha, teste frágil |
| S9 | Uso de `null` como retorno em vez de `Optional`/exceção | Assinatura de método retorna tipo simples mas pode retornar `null` | Propenso a `NullPointerException`, contrato implícito não documentado |
| S10 | Query complexa (`Specification`/`Criteria`) vazando para `application` | Import de tipo do Spring Data JPA fora de `adapters.out` | Acopla camada de aplicação a detalhe técnico de persistência |
| S11 | Textos de exceção ou valores literais duplicados ("Magic Strings") | Expressões de texto/mensagens de erro repetidas inline em múltiplos `throw` ou checagens | Dificulta manutenção, propenso a inconsistências de digitação e viola o princípio DRY |
| S12 | Quebras de linha arbitrárias e baixa densidade vertical | Métodos curtos, instâncias ou chamadas fluentes (Streams/Builders) fatiadas em várias linhas sem exceder 120 cols | Prejudica a leitura, fragmenta o código e consome espaço vertical desnecessariamente |

---

# PARTE B — Padrões do time (a ser preenchida)

Esta seção começa vazia. Conforme o time de desenvolvimento definir convenções próprias que **complementem ou sobrescrevam** a Parte A, registre aqui usando o template abaixo. Ao validar/implementar/revisar código, sempre confira esta seção **antes** de aplicar uma regra da Parte A — se houver entrada aqui, ela prevalece.

### Template para nova entrada

```
## <Nome do padrão/convenção>

- **Contexto**: em que situação se aplica.
- **Regra do time**: o que deve ser feito (substituindo ou complementando a Parte A).
- **Diverge da Parte A em**: [seção X] — motivo da divergência.
- **Exemplo**: trecho de código, se aplicável.
- **Data/decisão**: quando e por quem foi definido (ex.: ADR-00X, link de discussão).
```

*(nenhuma entrada registrada até o momento)*

---

## Resultado esperado
Código implementado conforme as regras das Partes A e B, sem nenhum dos code smells do checklist da seção 13, com cobertura de teste cobrindo todos os cenários de regra de negócio da especificação (caminho feliz, bordas, exceções). Em revisões, o resultado é um laudo apontando cada violação pelo identificador do checklist (ex.: S11) com o trecho afetado e a regra correspondente deste documento.

## Exemplos
Os exemplos de aplicação estão embutidos nas próprias seções de regra: formatação e extração de constantes na seção 1.2 (Java), factory method em `record` na seção 2.1, e interface de Strategy na seção 4.1. Ao revisar código, compare o trecho suspeito com esses exemplos antes de apontar a violação.

## Restrições
Não substituir uma regra da Parte A sem registro prévio na Parte B — se houver entrada na Parte B, ela prevalece. Não aplicar regras desta skill fora de projetos Java 25 + Spring do time. Não assumir que um padrão aqui descrito é obrigatório quando a seção marca opcionalidade (ex.: Facade, Virtual Threads, Chain of Responsibility); em caso de dúvida, prefira a solução mais simples (KISS). Não misturar com as regras de estrutura de pacotes — essas pertencem à skill `hexagonal-architecture`.
