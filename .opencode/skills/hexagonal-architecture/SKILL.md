---
name: hexagonal-architecture
description: Regras de arquitetura hexagonal (ports & adapters) adotadas no projeto Java/Spring — estrutura de pacotes, direção de dependências, onde ficam use cases, ports, adapters e DTOs. Use ao validar especificação técnica ou ao implementar/revisar código.
---

# Arquitetura Hexagonal — Regras de Validação

## Quando usar
Use ao validar especificação técnica (Fase 1), ao implementar código (Fase 2, junto da skill `design-patterns`), ou ao revisar código (Fase 3). É a referência única de "o que é correto" em arquitetura hexagonal (ports & adapters) neste projeto, destinada ao time de desenvolvimento e ao subagente `spec-validator`; qualquer validação, implementação ou review deve usar exatamente estas regras — não infira convenções alternativas.

## Instruções (passo a passo)
Na validação de especificação, siga a sequência da seção 8; na implementação, use as seções 2, 4 e 5 como guia de onde criar cada classe; no code review, reaplique o checklist da seção 7 sobre o código efetivamente implementado, não apenas sobre a spec. As regras detalhadas seguem nas seções numeradas abaixo.

## 1. Objetivo

Garantir que:
- O **domínio** (regras de negócio) não conheça framework, banco de dados ou protocolo de entrada/saída.
- A **direção de dependência** aponte sempre de fora para dentro (`adapters → application → domain`), nunca o contrário.
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
│   └── enums/              # Enums (Indicadores de status, validação e etc)           
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

Migrations Flyway ficam em `src/main/resources/db/migration`, fora da árvore de pacotes Java — não têm relação de dependência com o código, apenas com o schema.

---

## 3. A Regra de Dependência (Dependency Rule)

```
adapters/in  ──▶  application  ◀──  adapters/output
                        │
                        ▼
                     domain
```

- `domain` **não importa nada** de `application` ou `adapters`. Não conhece Spring, JPA, Jackson, nem HTTP.
- `application` importa apenas `domain`. Define **todas as Ports** (`port/input` para os casos de uso e `port/output` para o que precisa de infraestrutura) — nunca importa `adapters`.
- **Ports vivem exclusivamente em `application/port` (input e output) e são implementadas em `adapters/**`.** O `domain` não declara Ports e não tem relação com o "mundo" externo: ele não sabe que controllers, filas, banco ou HTTP existem — a aplicação orquestra o domínio e traduz suas necessidades para as Ports.
- `adapters/output` implementa as interfaces de `application/port/output`. É aqui que Spring/JPA/HTTP client aparecem.
- `adapters/input` depende de `application/port/input` (a interface do caso de uso), nunca da implementação (classe em `application/service`) diretamente, e nunca do `domain` puro sem passar pelo caso de uso.
- **Nenhuma seta pode apontar para dentro do domínio.** Se um `import` em `domain/**` referenciar qualquer classe de `adapters/**` ou `application/**`, isso é uma violação bloqueante.

---

## 4. Definições e convenções de nomenclatura

| Elemento | Pacote | Convenção de nome | Regras |
|---|---|---|---|
| Entidade de domínio | `domain.entity` | `Pedido`, `Cliente` | POJO puro; sem anotações JPA/Jackson/Spring; validações de invariante no construtor/métodos. |
| Value Object | `domain.valueobject` | `CPF`, `Dinheiro`, `Email` | Imutável; valida-se a si mesmo; `equals`/`hashCode` por valor. |
| Exceção de domínio | `domain.exception` | `PedidoInvalidoException`, `SaldoInsuficienteException` | Estende `RuntimeException` (ou base própria); representa violação de regra de negócio, não erro técnico. |
| Input Port (caso de uso) | `application.port.input` | `CriarPedidoPort` (interface) | Um método principal (`execute`/nome de intenção), assinatura em termos de domínio/DTO de aplicação, nunca em termos de `Entity` JPA ou DTO HTTP. |
| Implementação do caso de uso | `application.service` | `CriarPedidoService` | Implementa o Input Port; injeta apenas Output Ports (interfaces) e classes de `domain`; anotada com `@Service` (ainda é aplicação, mas Spring pode gerenciar o bean aqui). |
| Output Port | `application.port.output` | `PedidoPersistencePort`, `NotificacaoPort` | Interface; vive exclusivamente em `application.port.output` (nunca em `domain/**`); assinatura em termos de `domain.entity`/`domain.valueobject`, nunca `Entity` JPA. |
| Adapter de saída (persistência) | `adapters.output.infraestructure.persistence` | `PedidoPersistenceAdapter` | Implementa `PedidoPersistencePort`; injeta o `JpaRepository` e o `mapper`; único lugar que converte `Entity` JPA ↔ `domain.entity`. |
| Entidade JPA | `adapters.out.persistence.entity` | `PedidoEntity` | Anotada com `@Entity`/`@Table`; **nunca** retornada ou recebida por `application` ou `domain`. |
| Controller REST | `adapters.input.controller` | `PedidoController` | Injeta o **Input Port** (interface do caso de uso), nunca a implementação nem repositórios diretamente; converte request/response DTOs ↔ DTO de aplicação/domínio. |

---

## 5. Regras específicas de Spring / JPA / Flyway

1. **`@Entity`, `@Table`, `@Column`, `@Id` só existem em `adapters.output.persistence.entity`.** Se aparecerem em `domain/**`, é violação bloqueante ("vazamento de infraestrutura para o domínio").
2. **`@RestController`, `@RequestMapping`, `@Valid` em DTOs de request/response só existem em `adapters.in.web`.**
3. **`@Transactional`** deve estar na camada de `application.service` (orquestração da transação de negócio), não dentro do domínio, e idealmente não espalhado em múltiplos adapters.
4. **`@Service`/`@Component`/`@Repository`** em `domain/**` não deve existir — o domínio não é gerenciado pelo container Spring. `@Service` é aceitável em `application.service`; `@Repository`/`@Component` em `adapters.output`.
5. **Injeção de dependência sempre via interface**: a implementação do caso de uso nunca injeta `JpaRepository` diretamente — injeta a `OutputPort` (definida em `application.port.output`), cuja implementação (`*Adapter` em `adapters.output`) é resolvida pelo Spring.
6. **Migrations Flyway são imutáveis** após aplicadas (não editar `V{n}__*.sql` já mesclado); nova alteração = nova migration `V{n+1}__*.sql`. Migration não referencia lógica de aplicação, apenas DDL/DML de schema.
7. **Mapper domínio ↔ entidade JPA** deve existir explicitamente (classe `*Mapper` ou método dedicado) e viver em `adapters.output.infraestructure.persistence` — o domínio não sabe mapear-se para JPA.

---

## 6. Regras de testabilidade (reflexo da arquitetura)

- Testes de `domain` e `application.service` devem rodar **sem subir contexto Spring** (JUnit + Mockito puro), mockando apenas os Output Ports (interfaces). Se um teste de caso de uso precisar de `@SpringBootTest` ou de um banco real, isso é sinal de que o caso de uso está acoplado a infraestrutura.
- Testes de `adapters.output.persistence` podem usar `@DataJpaTest`/Testcontainers contra PostgreSQL real.
- Testes de `adapters.input.controller` podem usar `@WebMvcTest`, mockando o Input Port.

---

## 7. Checklist de violações comuns

| # | Violação | Como detectar (na especificação ou no código) | Por que é um problema |
|---|---|---|---|
| V1 | Entidade de domínio anotada com `@Entity`/`@Table` | Classe em `domain/**` (ou descrita como tal na spec) contém anotações JPA | Acopla regra de negócio ao mecanismo de persistência; troca de banco/ORM exige alterar o domínio |
| V2 | Controller acessando `JpaRepository` ou `PersistenceAdapter` diretamente | Import de `adapters.output.*` dentro de `adapters.input.controller` | Pula a camada de aplicação; a regra de negócio deixa de ser orquestrada pelo caso de uso, quebrando a fronteira do hexágono |
| V3 | Caso de uso recebendo/retornando `Entity` JPA | Assinatura de método do Input/Output Port referencia classe de `adapters.output.infraestructure.persistence.entity` | Application passa a depender de detalhe de infraestrutura; muda o schema, quebra a assinatura do caso de uso |
| V4 | Output Port ausente — Use Case chama `JpaRepository` diretamente | Ausência de interface em `application.port.output` para uma dependência externa citada na spec | Impede troca/mocking do adapter e testes unitários do caso de uso sem infraestrutura |
| V5 | Regra de negócio implementada dentro do Controller ou do Adapter de persistência | Spec descreve validação/decisão de negócio ocorrendo em `adapters.input` ou `adapters.output` | Regra de negócio duplicada/fragmentada fora do domínio, difícil de testar e reaproveitar |
| V6 | Exceção técnica (`SQLException`, `DataAccessException`) vazando até o domínio ou a resposta HTTP sem tradução | Spec não menciona um `ExceptionHandler`/tradução de exceção na borda | Usuário final ou camada de domínio passam a depender de exceções específicas de infraestrutura |
| V7 | Migration Flyway alterando dado de forma acoplada a uma versão específica de entidade Java | Migration referencia diretamente nomes de classes/DTOs Java | Acopla schema à implementação, quebra em refactors futuros |
| V8 | Múltiplas responsabilidades num único Use Case ("Deus caso de uso") | Um único Input Port descrito na spec cobre vários casos de uso distintos (ex.: criar, atualizar e cancelar pedido no mesmo método) | Fere responsabilidade única, dificulta teste e reuso |
| V9 | Domain Service com dependência de bean Spring (`@Autowired` em classe de `domain.service`) | Import de `org.springframework.*` dentro de `domain/**` | Rompe a regra de dependência (domínio não pode depender de framework) |
| V10 | DTO de request/response HTTP reaproveitado como objeto de domínio | Mesmo objeto usado tanto na assinatura do Controller quanto do caso de uso/domínio | Acopla contrato de API ao modelo interno; mudança de contrato externo força mudança no domínio |
| V11 | Port (Input ou Output) declarada fora de `application.port` — inclusive em `domain/**` | Interface de port em `domain.ports`, `domain.ports.output` ou qualquer pacote que não seja `application.port.input`/`application.port.output` | Rompe a regra de dependência: Ports pertencem exclusivamente à Application e são implementadas nos Adapters; Ports no domínio criam acoplamento dele ao mundo externo |

---

## 8. Processo de validação (para o subagente `spec-validator`)

Ao validar uma especificação técnica, siga esta sequência:

1. **Mapear os elementos citados na especificação** para as camadas da seção 2 (domain / application / adapters). Se um elemento não puder ser classificado claramente em uma camada, isso já é um sinal de ambiguidade a reportar.
2. **Verificar a direção das dependências** descritas ou implícitas na spec (quem chama quem, quem injeta o quê) contra a seção 3. Qualquer seta invertida é violação bloqueante.
3. **Rodar o checklist da seção 7** item a item contra o conteúdo da spec.
4. **Verificar nomenclatura** (seção 4) — divergências de nome não são bloqueantes por si só, mas devem ser reportadas como inconsistência a alinhar com o desenvolvedor.
5. Para cada violação encontrada, reportar no formato:

```
- **Violação**: [V<n> — descrição curta]
  **Onde**: [trecho/seção da especificação]
  **Princípio violado**: [regra da seção 3/5]
  **Por que é um problema**: [consequência concreta]
  **Sugestão** (opcional, não aplicar automaticamente): [ajuste recomendado]
```

6. Nunca "corrigir" a especificação nesta fase — apenas reportar. A decisão de ajustar a spec é do desenvolvedor.

---

## Resultado esperado
Depende da fase: na Fase 1 (validação), um laudo listando cada violação encontrada com o formato da seção 8 (item 5), sem alterar a especificação; na Fase 2 (implementação), código com todas as classes nos pacotes e com os nomes definidos nas seções 2 e 4; na Fase 3 (code review), checklist da seção 7 aplicado integralmente, com cada violação apontada pelo identificador (ex.: V4) e o trecho afetado. O resultado só é considerado aprovado quando nenhuma violação bloqueante (setas apontando para dentro do domínio, anotações de infraestrutura no domínio, Entity JPA em assinatura de Port) permanece.

---

## Exemplos — correto vs. incorreto

**Incorreto** (Output Port ausente, Use Case acoplado à entidade JPA):

```java
// application/service/CriarPedidoService.java
@Service
public class CriarPedidoService implements CriarPedidoUseCase {
    private final PedidoJpaRepository jpaRepository; // ❌ depende de infra direto (sem Output Port)

    public PedidoEntity executar(PedidoRequest request) { // ❌ DTO HTTP e Entity JPA na assinatura
        PedidoEntity entity = new PedidoEntity(...);
        return jpaRepository.save(entity);
    }
}
```

**Correto**:

```java
// application/port/output/PedidoPersistencePort.java
public interface PedidoPersistencePort {
    Pedido salvar(Pedido pedido); // trabalha com o modelo de domínio
}

// application/service/CriarPedidoService.java
@Service
public class CriarPedidoService implements CriarPedidoUseCase {
    private final PedidoPersistencePort persistencePort; // ✅ depende da interface da Port (application)

    public Pedido executar(CriarPedidoCommand command) {
        Pedido pedido = Pedido.criar(command.clienteId(), command.itens());
        return persistencePort.salvar(pedido);
    }
}

// adapters/output/infrastructure/persistence/PedidoPersistenceAdapter.java
@Component
public class PedidoPersistenceAdapter implements PedidoPersistencePort {
    private final PedidoJpaRepository jpaRepository;
    private final PedidoMapper mapper;

    public Pedido salvar(Pedido pedido) {
        PedidoEntity entity = mapper.paraEntity(pedido);
        return mapper.paraDominio(jpaRepository.save(entity));
    }
}
```

---

## Restrições
Nunca "corrigir" a especificação durante a validação — apenas reportar; a decisão de ajustar a spec é do desenvolvedor. Não infer convenções alternativas não previstas neste documento. Não tratar divergências de nomenclatura como bloqueantes — reportá-las como inconsistência a alinhar. Não definir regras de *como* implementar dentro de cada camada (padrões de projeto, testes, exceções) — essas pertencem à skill `design-patterns`.