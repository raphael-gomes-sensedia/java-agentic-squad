---
name: hexagonal-architecture
description: Regras de arquitetura hexagonal (ports & adapters) adotadas no projeto Java/Spring — estrutura de pacotes, direção de dependências, onde ficam use cases, ports, adapters e DTOs. Use ao validar especificação técnica ou ao implementar/revisar código.
---

# Arquitetura Hexagonal — Regras de Validação

Este documento é a referência única de "o que é correto" em termos de arquitetura
hexagonal (ports & adapters) neste projeto. Qualquer validação, implementação ou
review deve usar exatamente estas regras — não infira convenções alternativas.

## 1. Objetivo

Garantir que:
- O **domínio** (regras de negócio) não conheça framework, banco de dados ou
  protocolo de entrada/saída.
- A **direção de dependência** aponte sempre de fora para dentro
  (`adapters → application → domain`), nunca o contrário.
- Cada camada tenha responsabilidade única e testável isoladamente.

---

## 2. Estrutura de pacotes esperada

```
src/main/java/com/empresa/projeto/
│
├── domain/
│   ├── entity/             # Entidades de domínio (POJOs puros, sem @Entity)
│   ├── valueobject/        # Value Objects (imutáveis, com validação própria)
│   ├── exception/          # Exceções de domínio (regras de negócio violadas)
│   ├── enums/              # Enums (Indicadores de status, validação e etc)
│   └── ports/             
│       └── output/         # Output Ports = interfaces de infra para realizar operações da lógica de negócio
│
├── application/
│   ├── port/
│   │   ├── input/             # Input Ports = interfaces dos casos de uso (Port)
│   │   └── output/            # Output Ports = interfaces que o domínio/app precisa da infra
│   ├── service/               # Implementações dos Input Ports (orquestram o domínio)
│   └── commom/                # Classes para controle de status e validações para o fluxo
│
├── adapters/
│   ├── input/
│   │   ├── controller/      # Controllers REST, request/response DTOs, exception handlers, mappers
|   |   |   └── dto/
|   |   |   └── handler/
|   |   |   └── mapper/   
│   │   └── messaging/       # Listeners de fila/eventos (se houver)
│   └── output/
│   │   ├── email/
│   │   ├── externalservices/
│   │   └── infraestructure/
│   │      ├── persistence/         # Tudo que diz respeito as adapters para persistencia no banco
│   │      │   ├── entity/          # Entidades JPA (@Entity) — NUNCA usadas fora deste pacote
│   │      │   ├── repository/      # Spring Data JpaRepository (interfaces)
│   │      │   ├── converter/       # Converter de JSONs persistidos no banco
│   │      │   └── mapper/          # Mappers de Domain para Entity e vice versa
│   │      ├── httpclient/          # Clients HTTP/gRPC para sistemas externos
│   │      └── logs/            
│   └── config/                     # Configuração Spring (beans, security, rest, jwt, logs etc.)
│
├── config/         
└── utils/          # Classes de utilidade                  
```

Migrations Flyway ficam em `src/main/resources/db/migration`, fora da árvore de
pacotes Java — não têm relação de dependência com o código, apenas com o schema.

---

## 3. A Regra de Dependência (Dependency Rule)

```
adapters/in  ──▶  application  ◀──  adapters/output
                        │
                        ▼
                     domain
```

- `domain` **não importa nada** de `application` ou `adapters`. Não conhece Spring,
  JPA, Jackson, nem HTTP.
- `application` importa apenas `domain`. Define **interfaces** (`port/output`) para o
  que precisa de infraestrutura — nunca importa `adapters`.
- `adapters/output` implementa as interfaces de `application/port/output`. É aqui que
  Spring/JPA/HTTP client aparecem.
- `adapters/input` depende de `application/port/input` (a interface do caso de uso), nunca
  da implementação (`usecase.XyzUseCaseImpl`) diretamente, e nunca do `domain` puro
  sem passar pelo caso de uso.
- **Nenhuma seta pode apontar para dentro do domínio.** Se um `import` em
  `domain/**` referenciar qualquer classe de `adapters/**` ou `application/**`,
  isso é uma violação bloqueante.

---

## 4. Definições e convenções de nomenclatura

| Elemento | Pacote | Convenção de nome | Regras |
|---|---|---|---|
| Entidade de domínio | `domain.entity` | `Pedido`, `Cliente` | POJO puro; sem anotações JPA/Jackson/Spring; validações de invariante no construtor/métodos. |
| Value Object | `domain.valueobject` | `CPF`, `Dinheiro`, `Email` | Imutável; valida-se a si mesmo; `equals`/`hashCode` por valor. |
| Exceção de domínio | `domain.exception` | `PedidoInvalidoException`, `SaldoInsuficienteException` | Estende `RuntimeException` (ou base própria); representa violação de regra de negócio, não erro técnico. |
| Input Port (caso de uso) | `application.port.input` | `CriarPedidoPort` (interface) | Um método principal (`execute`/nome de intenção), assinatura em termos de domínio/DTO de aplicação, nunca em termos de `Entity` JPA ou DTO HTTP. |
| Implementação do caso de uso | `application.port` | `CriarPedidoSerivce` | Implementa o Input Port; injeta apenas Output Ports (interfaces) e classes de `domain`; anotada com `@Service` (ainda é aplicação, mas Spring pode gerenciar o bean aqui). |
| Output Port | `application.port.output` | `PedidoPersistencePort`, `NotificacaoPort` | Interface; assinatura em termos de `domain.model`, nunca `Entity` JPA. |
| Adapter de saída (persistência) | `adapters.output.infraestructure.persistence` | `PedidoPersistenceAdapter` | Implementa `PedidoRepositoryPort`; injeta o `JpaRepository` e o `mapper`; único lugar que converte `Entity` JPA ↔ `domain.model`. |
| Entidade JPA | `adapters.out.persistence.entity` | `PedidoEntity` | Anotada com `@Entity`/`@Table`; **nunca** retornada ou recebida por `application` ou `domain`. |
| Controller REST | `adapters.input.controller` | `PedidoController` | Injeta o **Input Port** (interface do caso de uso), nunca a implementação nem repositórios diretamente; converte request/response DTOs ↔ DTO de aplicação/domínio. |

---

## 5. Regras específicas de Spring / JPA / Flyway

1. **`@Entity`, `@Table`, `@Column`, `@Id` só existem em `adapters.output.persistence.entity`.**
   Se aparecerem em `domain.model`, é violação bloqueante ("vazamento de
   infraestrutura para o domínio").
2. **`@RestController`, `@RequestMapping`, `@Valid` em DTOs de request/response só
   existem em `adapters.in.web`.**
3. **`@Transactional`** deve estar na camada de `application.usecase` (orquestração
   da transação de negócio), não dentro do domínio, e idealmente não espalhado em
   múltiplos adapters.
4. **`@Service`/`@Component`/`@Repository`** em `domain/**` não deve existir — o
   domínio não é gerenciado pelo container Spring. `@Service` é aceitável em
   `application.usecase`; `@Repository`/`@Component` em `adapters.output`.
5. **Injeção de dependência sempre via interface**: um `UseCaseImpl` nunca injeta
   `JpaRepository` diretamente — injeta o `OutputPort`, cuja implementação
   (`PersistenceAdapter`) é resolvida pelo Spring.
6. **Migrations Flyway são imutáveis** após aplicadas (não editar `V{n}__*.sql` já
   mesclado); nova alteração = nova migration `V{n+1}__*.sql`. Migration não
   referencia lógica de aplicação, apenas DDL/DML de schema.
7. **Mapper domínio ↔ entidade JPA** deve existir explicitamente (classe `*Mapper`
   ou método dedicado) e viver em `adapters.output.infraestructure.persistence` — o domínio não sabe
   mapear-se para JPA.

---

## 6. Regras de testabilidade (reflexo da arquitetura)

- Testes de `domain` e `application.service` devem rodar **sem subir contexto
  Spring** (JUnit + Mockito puro), mockando apenas os Output Ports (interfaces).
  Se um teste de caso de uso precisar de `@SpringBootTest` ou de um banco real,
  isso é sinal de que o caso de uso está acoplado a infraestrutura.
- Testes de `adapters.output.persistence` podem usar `@DataJpaTest`/Testcontainers
  contra PostgreSQL real.
- Testes de `adapters.input.controller` podem usar `@WebMvcTest`, mockando o Input Port.

---

## 7. Checklist de violações comuns

| # | Violação | Como detectar (na especificação ou no código) | Por que é um problema |
|---|---|---|---|
| V1 | Entidade de domínio anotada com `@Entity`/`@Table` | Classe em `domain.model` (ou descrita como tal na spec) contém anotações JPA | Acopla regra de negócio ao mecanismo de persistência; troca de banco/ORM exige alterar o domínio |
| V2 | Controller acessando `JpaRepository` ou `PersistenceAdapter` diretamente | Import de `adapters.output.*` dentro de `adapters.input.controller` | Pula a camada de aplicação; a regra de negócio deixa de ser orquestrada pelo caso de uso, quebrando a fronteira do hexágono |
| V3 | Caso de uso recebendo/retornando `Entity` JPA | Assinatura de método do Input/Output Port referencia classe de `adapters.output.infraestructure.persistence.entity` | Application passa a depender de detalhe de infraestrutura; muda o schema, quebra a assinatura do caso de uso |
| V4 | Output Port ausente — Use Case chama `JpaRepository` diretamente | Ausência de interface em `application.port.output` para uma dependência externa citada na spec | Impede troca/mocking do adapter e testes unitários do caso de uso sem infraestrutura |
| V5 | Regra de negócio implementada dentro do Controller ou do Adapter de persistência | Spec descreve validação/decisão de negócio ocorrendo em `adapters.input` ou `adapters.output` | Regra de negócio duplicada/fragmentada fora do domínio, difícil de testar e reaproveitar |
| V6 | Exceção técnica (`SQLException`, `DataAccessException`) vazando até o domínio ou a resposta HTTP sem tradução | Spec não menciona um `ExceptionHandler`/tradução de exceção na borda | Usuário final ou camada de domínio passam a depender de exceções específicas de infraestrutura |
| V7 | Migration Flyway alterando dado de forma acoplada a uma versão específica de entidade Java | Migration referencia diretamente nomes de classes/DTOs Java | Acopla schema à implementação, quebra em refactors futuros |
| V8 | Múltiplas responsabilidades num único Use Case ("Deus caso de uso") | Um único Input Port descrito na spec cobre vários casos de uso distintos (ex.: criar, atualizar e cancelar pedido no mesmo método) | Fere responsabilidade única, dificulta teste e reuso |
| V9 | Domain Service com dependência de bean Spring (`@Autowired` em classe de `domain.service`) | Import de `org.springframework.*` dentro de `domain/**` | Rompe a regra de dependência (domínio não pode depender de framework) |
| V10 | DTO de request/response HTTP reaproveitado como objeto de domínio | Mesmo objeto usado tanto na assinatura do Controller quanto do caso de uso/domínio | Acopla contrato de API ao modelo interno; mudança de contrato externo força mudança no domínio |

---

## 8. Processo de validação (para o subagente `spec-validator`)

Ao validar uma especificação técnica, siga esta sequência:

1. **Mapear os elementos citados na especificação** para as camadas da seção 2
   (domain / application / adapters). Se um elemento não puder ser classificado
   claramente em uma camada, isso já é um sinal de ambiguidade a reportar.
2. **Verificar a direção das dependências** descritas ou implícitas na spec
   (quem chama quem, quem injeta o quê) contra a seção 3. Qualquer seta invertida
   é violação bloqueante.
3. **Rodar o checklist da seção 7** item a item contra o conteúdo da spec.
4. **Verificar nomenclatura** (seção 4) — divergências de nome não são bloqueantes
   por si só, mas devem ser reportadas como inconsistência a alinhar com o
   desenvolvedor.
5. Para cada violação encontrada, reportar no formato:

```
- **Violação**: [V<n> — descrição curta]
  **Onde**: [trecho/seção da especificação]
  **Princípio violado**: [regra da seção 3/5]
  **Por que é um problema**: [consequência concreta]
  **Sugestão** (opcional, não aplicar automaticamente): [ajuste recomendado]
```

6. Nunca "corrigir" a especificação nesta fase — apenas reportar. A decisão de
   ajustar a spec é do desenvolvedor.

---

## 9. Exemplo — correto vs. incorreto

**Incorreto** (Output Port ausente, Use Case acoplado à entidade JPA):

```java
// application/usecase/CriarPedidoUseCaseImpl.java
@Service
public class CriarPedidoUseCaseImpl implements CriarPedidoUseCase {
    private final PedidoJpaRepository jpaRepository; // ❌ depende de infra direto

    public PedidoEntity executar(PedidoRequest request) { // ❌ DTO HTTP e Entity JPA na assinatura
        PedidoEntity entity = new PedidoEntity(...);
        return jpaRepository.save(entity);
    }
}
```

**Correto**:

```java
// application/port/out/PedidoRepositoryPort.java
public interface PedidoRepositoryPort {
    Pedido salvar(Pedido pedido); // trabalha com o modelo de domínio
}

// application/usecase/CriarPedidoUseCaseImpl.java
@Service
public class CriarPedidoUseCaseImpl implements CriarPedidoUseCase {
    private final PedidoRepositoryPort repositoryPort; // ✅ depende de interface

    public Pedido executar(CriarPedidoCommand command) {
        Pedido pedido = Pedido.criar(command.clienteId(), command.itens());
        return repositoryPort.salvar(pedido);
    }
}

// adapters/out/persistence/adapter/PedidoPersistenceAdapter.java
@Component
public class PedidoPersistenceAdapter implements PedidoRepositoryPort {
    private final PedidoJpaRepository jpaRepository;
    private final PedidoMapper mapper;

    public Pedido salvar(Pedido pedido) {
        PedidoEntity entity = mapper.paraEntity(pedido);
        return mapper.paraDominio(jpaRepository.save(entity));
    }
}
```

---

## 10. Quando aplicar esta skill

- **Fase 1 (validação)**: comparar a especificação técnica contra todas as seções
  acima antes de liberar a implementação.
- **Fase 2 (implementação/TDD)**: usar as seções 2, 4 e 5 como guia de onde criar
  cada classe.
- **Fase 3 (code review)**: reaplicar o checklist da seção 7 sobre o código
  efetivamente implementado, não apenas sobre a spec.