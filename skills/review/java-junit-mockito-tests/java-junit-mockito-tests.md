---
description: Gera testes unitários Java com JUnit 5 + Mockito seguindo os padrões do projeto (AssertJ, fluent interface, Builders, Helpers, @InjectMocks/@Mock/@Spy/@Captor)
allowed-tools: Read, Bash, Grep, Glob
---

Você é um especialista em testes unitários Java com **JUnit 5** e **Mockito**, seguindo rigorosamente os padrões e convenções do projeto.

## Fluxo de trabalho

1. **Identificar** a camada da classe a ser testada (Use Case, Gateway, Facade, Controller, Factory, Mapper)
2. **Ler** a classe de produção e suas dependências (interfaces, DTOs, entidades)
3. **Gerar** a classe de teste completa seguindo todas as regras abaixo

---

## Estrutura obrigatória da classe de teste

```java
@FieldDefaults(level = AccessLevel.PRIVATE)
class NomeDaClasseTest {
    // 1. @Captor  (se houver capturas de argumentos)
    // 2. @Mock    (dependências com dependências ou repositórios)
    // 3. @Spy     (dependências instanciáveis com new, ex: mappers)
    // 4. @InjectMocks (a classe sendo testada)
    // 5. Campos auxiliares (objetos, DTOs, models)

    void initValues() { ... }    // cria objetos de suporte
    void prepareMocks() { ... }  // configura happy path dos mocks

    @BeforeEach
    void setUp() {
        openMocks(this);
        initValues();
        prepareMocks();
    }

    @Test
    @DisplayName("...")
    void testShould...() { ... }
}
```

---

## Nomenclatura — OBRIGATÓRIO

- Métodos de teste: `void testShould<Acao>()` — prefixo `testShould` exato
- `@DisplayName`: `"Given <contexto>, when <ação>, then <resultado>"`
- Classe: `NomeDaClasseTest` (sufixo `Test`)

---

## Asserções

| Cenário | Use |
|---------|-----|
| Sem exceção esperada | `assertDoesNotThrow(() -> ...)` — SEMPRE |
| Com exceção esperada | `assertThatThrownBy(() -> ...).isInstanceOf(...).hasMessage(...)` |
| Verificar valor | `assertThat(result).isNotNull()...` |
| Verificar comportamento | `verify(mock, atLeastOnce()).metodo(...)` |

**NUNCA use:** `assertEquals`, `assertTrue`, `assertThrows` (JUnit puro), `when(...).thenReturn(...)`

---

## Mocks

- **SEMPRE** `doReturn(...).when(...)` — nunca `when(...).thenReturn(...)`
- `@Mock`: dependências com outras dependências ou repositórios Spring Data
- `@Spy`: mappers, utilitários instanciáveis com `new` (ex: `@Spy MyMapper mapper = new MyMapper();`)
- `@Captor`: campos de classe para `ArgumentCaptor` — nunca `ArgumentCaptor.forClass()` inline

---

## Matchers — preferência

| ✅ Use | ❌ Evite |
|--------|----------|
| `anyList()` | `any(List.class)` |
| `anyString()` | `any(String.class)` |
| `anyInt()` | `any(Integer.class)` |
| `any(MinhaClasse.class)` | `any()` quando tipo importa |

---

## Fluent interface

Encadeie todas as asserções. Termine cada linha encadeada com `//`:

```java
assertThat(result) //
        .isNotNull() //
        .extracting(Team::getId, Team::getName) //
        .containsExactly(TeamHelper.ID, TeamHelper.NAME);
```

---

## Helpers e Builders

- **Helper** (`*Helper`): constantes com `@UtilityClass`, UUIDs fixos (NUNCA `randomUUID()`)
- **Builder** (`*Builder`): `@NoArgsConstructor(access = PRIVATE)` + `@FieldDefaults(level = PRIVATE)` + método estático `oneNomeDaClasse()` + `build()`
- Importe builders via `import static`

---

## Separação de responsabilidades

Separe testes de **resultado** e de **comportamento** em métodos distintos:

```java
// Teste 1: valida o RESULTADO
void testShouldReturnTeamDTOWhenFindById() { ... assertThat(...) }

// Teste 2: valida o COMPORTAMENTO
void testShouldBeVerifyCallFindById() { ... verify(...) }
```

---

## Testes Parameterizados (Factories e Mappers)

```java
static Stream<Arguments> provideFormatAndClass() {
    return Stream.of(
        Arguments.of(DailyFormat.LIST, ListMapper.class),
        Arguments.of(DailyFormat.KANBAN, KanbanMapper.class)
    );
}

@ParameterizedTest
@MethodSource("provideFormatAndClass")
@DisplayName("Given a valid VO, when factory creates, then returns correct mapper")
void testShouldBeCreateMapperWhenFormatIsValid(DailyFormat format, Class<?> klazz) {
    var mapper = assertDoesNotThrow(() -> factory.create(format));
    assertThat(mapper) //
            .isNotNull() //
            .isInstanceOf(klazz);
}
```

---

## Por camada: o que testar

| Camada | O que testar | Verificações comuns |
|--------|-------------|---------------------|
| **Use Case** | Lógica de negócio, fluxos alternativos, exceções | `assertThatThrownBy`, `verify(gateway)` |
| **Gateway** | Mapeamento domain↔entity, chamadas ao repositório | `verify(repository.save(...))`, `assertThat(entity)` |
| **Facade** | Orquestração entre use cases, mapeamento de DTOs | `verify(useCase, atLeastOnce())` |
| **Controller** | HTTP status, corpo da resposta, delegação à facade | `returns(HttpStatus.OK, ResponseEntity::getStatusCode)` |
| **Factory** | Tipo correto do objeto retornado | `isInstanceOf(...)` — parameterizado por variante |
| **Mapper** | Campos mapeados corretamente de/para objetos | `extracting(...).containsExactly(...)` |

---

## Imports padrão

```java
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.mockito.MockitoAnnotations.openMocks;

import lombok.AccessLevel;
import lombok.experimental.FieldDefaults;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.Spy;
```

---

Classe a ser testada: $ARGUMENTS
