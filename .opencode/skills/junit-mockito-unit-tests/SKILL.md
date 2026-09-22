---
name: junit-mockito-unit-tests
description: Definições do time para criação de testes unitários com JUnit 5 + Mockito — nomenclatura, estrutura AAA, organização dos mocks e testes mutáveis. Use ao escrever, completar ou corrigir testes unitários de use cases, serviços e classes de domínio em projetos backend Java 25 + Spring do time.
---

# Testes Unitários JUnit 5 + Mockito

## Quando usar
- Sempre que for necessário criar, completar ou ajustar testes unitários no backend Java 25 + Spring + PostgreSQL + Flyway + JUnit/Mockito do time, respeitando a arquitetura hexagonal.
- Aplica-se a testes de `application.usecase` (com Mockito), `domain` (JUnit puro, sem Mockito nem Spring) e quando os testes surgem DEPOIS da implementação (fluxo implementação → testes, neste projeto).

## Instruções (passo a passo)

1. **Leia primeiro a implementação e a especificação**: neste projeto o código de produção já existe quando os testes são escritos. Derive dos métodos/casos de uso implementados e da especificação um checklist de cenários a cobrir: caminho feliz, bordas e exceções. Cubra TODO cenário de regra de negócio da especificação — cobertura numérica é consequência, não objetivo.
2. **Nomenclatura**:
   - Classe de teste: nome da classe sob teste + sufixo `Test` (ex.: `ProcessarPagamentoUseCaseTest`), no MESMO pacote da classe testada (espelhamento de pacote em `src/test/java`).
   - Método de teste: nome descritivo do comportamento, no formato `deveXxx_quandoYyy()` (ex.: `deveLancarExcecao_quandoPedidoSemItens()`) ou BDD-style `xxx_comYyy_deveFazerZzz()` — escolha UM estilo e use em todo o projeto, sem misturar.
   - Não use números (`test1`, `teste2`) nem nomes genéricos (`validarTest`); cada método explica o cenário sem precisar abrir o corpo.
   - Campos/fixtures com nomes de domínio (`pagamentoValido`, `clienteInexistente`), jamais `obj1`, `dto2`.
3. **Estrutura AAA (Arrange / Act / Assert)** — obrigatória em todo teste:
   - **Arrange**: prepare o mundo do teste (builders de dados, stubs de mocks). Somente o necessário para este cenário.
   - **Act**: EXATAMENTE UMA chamada ao método sob teste (armazene retorno em variável quando for asserido).
   - **Assert**: valide o resultado (estado/retorno) e, quando relevante, a interação essencial.
   - Separe as três seções com linha em branco, precedidas por comentários `// Arrange`, `// Act`, `// Assert`.
   - **Um comportamento por teste**: proibido cobrir comportamentos distintos no mesmo método; asserções múltiplas só são aceitáveis se verificarem facets do MESMO comportamento.
4. **Organização dos mocks**:
   - Estilo padrão: `@ExtendWith(MockitoExtension.class)` + `@Mock` para cada Output Port (interface) + `@InjectMocks` para o use case sob teste. Alternativa aceita (prefira em testes novos): instanciação explícita do use case via construtor com mocks criados em campos/fixture — explora a injeção por construtor do projeto e evita reflexo mágico.
   - **Mocke apenas os Output Ports** (interfaces de `adapters.out` / ports). NUNCA mocke classes de domínio, Value Objects (`record`) ou DTOs — instancie-os de verdade (mock de classe de domínio é code smell S5).
   - Stubbing enxuto: defina apenas os stubs que o cenário usa; o `Strictness.STRICT_STUBS` (padrão do MockitoExtension) acusará stubs desnecessários — não use `lenient()` para "silenciar" o excesso, remova o stub supérfluo.
   - Agrupe os stubs por método de teste (arrange de cada teste) — evite stubbing em `@BeforeEach` para comportamentos específicos de um único cenário.
   - `verify` com parcimônia: verifique SOMENTE interações essenciais ao comportamento testado (ex.: `verify(pedidoRepositoryPort).salvar(pedidoSalvo)`); proibido over-verification de chamadas incidentais.
   - Para capturar objetos modificados pela chamada, use `ArgumentCaptor` em vez de qualquer-match (`any()`) seguido de asserção vaga.
   - Se um teste exige mais que 5–6 mocks, pare e sinalize: provável violação de SRP no use case (reporte, não disfarce).
5. **Dados de teste mutáveis (Test Data Builder / Object Mother)**:
   - Crie builders (ex.: `PagamentoTestDataBuilder`) com valores-padrão válidos e métodos `with*` que retornam NOVA instância (compatível com `record`), permitindo variar apenas o que o cenário importa (ex.: `.comValorInvalido()`, `.comStatusExpirado()`).
   - Nunca duplique a montagem completa de objetos em cada teste: todos os testes partem do builder com defaults + variação pontual.
   - Object Mother fica para famílias fixas de dados reutilizadas entre classes de teste (ex.: `DominioDeTeste.pagamentoAprovado()`); builder para variações de cenário.
6. **Testes mutáveis (prontos para mutação/mutation testing)** — cada teste deve MORRER se o comportamento deixar de existir:
   - Asserção SEMPRE sobre valores esperados explícitos, nunca apenas "não nulo" / "não lança" quando a regra exige valor exato.
   - Numéricos, textos de mensagem de exceção e campos derivados: asserção exata (ex.: `assertEquals(new BigDecimal("29.90"), total)`, não `assertTrue(total > ZERO)` quando o resultado é calculado).
   - Valide tanto o RETORNO quanto o estado capturado do lado de fora (porta salva) quando ambos compõem o comportamento — mutar o objeto passado adiante deve derrubar o teste.
   - Proibido asserção "elástica" que passa mesmo com o código mutado (ex.: `assertThat(resultado).isNotNull()` em teste cujo requisito era somar juros).
   - Em `switch`/ramificações cobertas, cada ramo relevante da regra tem teste próprio (mutação de uma ramificação não pode sobreviver).
7. **Domínio puro** (`domain/**`): teste clássico de unidade — JUnit 5 sem Mockito nem Spring, usando builders/records; valide invariantes de factory, exceções de domínio e comportamento puro (`assertThrows` para exceções, comparando mensagem/propriedades relevantes).
8. **Execute** `mvn test` e repita até TODOS os testes passarem. Se um teste falhar e a investigação mostrar DEFEITO no código de produção, não corrija a produção por conta própria — sinalize (os subagentes definem a conduta; normalmente é perguntar ao desenvolvedor).
9. **Não modifique a implementação para "facilitar o teste"** e não enfraqueça a regra 6: ajuste o teste, nunca o requisito.

## Resultado esperado
Classes de teste espelhando o pacote da classe sob teste, com nomenclatura `*Test`, métodos descritivos no padrão único do time, corpo dividido em `// Arrange // Act // Assert` com linha em branco entre seções, mocks restritos a Output Ports com stubbing enxuto, builders de dados de teste com `with*` reutilizados por todos os cenários, e asserções exatas suficientes para matar mutações plausíveis da regra testada. Todos os testes verdes após `mvn test`.

## Exemplos

Esqueleto de teste de use case com mocks organizados, AAA e captor:

```java
@ExtendWith(MockitoExtension.class)
class ProcessarPagamentoUseCaseTest {

    @Mock
    private PagamentoRepositoryPort pagamentoRepositoryPort;

    @InjectMocks
    private ProcessarPagamentoUseCase useCase;

    @Test
    void deveSalvarPagamento_quandoClientePossuiCreditoDisponivel() {
        // Arrange
        var comando = new PagamentoTestDataBuilder().comValor("29.90").build();
        when(pagamentoRepositoryPort.buscarCredito(comando.clienteId())).thenReturn(Credito.disponivel());

        // Act
        var resultado = useCase.executar(comando);

        // Assert
        assertEquals(StatusPagamento.APROVADO, resultado.status());
        var captor = ArgumentCaptor.forClass(Pagamento.class);
        verify(pagamentoRepositoryPort).salvar(captor.capture());
        assertEquals(new BigDecimal("29.90"), captor.getValue().valor());
    }

    @Test
    void deveLancarPagamentoInvalidoException_quandoValorMenorQueZero() {
        // Arrange
        var comando = new PagamentoTestDataBuilder().comValor("-1").build();

        // Act + Assert
        var excecao = assertThrows(PagamentoInvalidoException.class, () -> useCase.executar(comando));
        assertEquals(Pagamento.MSG_VALOR_INVALIDO, excecao.getMessage());
        verify(pagamentoRepositoryPort, never()).salvar(any());
    }
}
```

Exemplo de builder mutável:

```java
public class PagamentoTestDataBuilder {

    private BigDecimal valor = new BigDecimal("100.00");
    private ClienteId clienteId = new ClienteId("CLI-1");
    private StatusPagamento status = StatusPagamento.PENDENTE;

    public PagamentoTestDataBuilder comValor(String valor) { this.valor = new BigDecimal(valor); return this; }

    public PagamentoTestDataBuilder comStatus(StatusPagamento status) { this.status = status; return this; }

    public Pagamento build() { return new Pagamento(clienteId, valor, status); }
}
```

## Restrições
- Não cubre estrutura de pacotes nem fluxo de camadas — isso pertence às skills `hexagonal-architecture` e `design-patterns`; em conflito com elas, prevalece o que for mais específico a testes nesta skill.
- Não aplique fora de projetos Java 25 + Spring do time; não use JUnit 4, `@RunWith` ou PowerMock.
- Não substitua regras da Parte B de `design-patterns` sem registrar lá a divergência.
- Não integre Spring Boot no teste unitário de use case/domain (isso é território de teste de integração, fora do escopo desta skill).
