# java-junit-mockito-tests

Generates complete Java unit test classes using JUnit 5 + Mockito, following strict project conventions: AssertJ fluent assertions, `@InjectMocks`/`@Mock`/`@Spy`/`@Captor`, Builders, Helpers, and separated result vs. behavior tests.

## Trigger

```
/java-junit-mockito-tests <ClassName or file path>
```

## What it does

1. Identifies the layer of the class to be tested (Use Case, Gateway, Facade, Controller, Factory, Mapper)
2. Reads the production class and its dependencies (interfaces, DTOs, entities)
3. Generates a complete test class following all rules:
   - `@FieldDefaults(level = AccessLevel.PRIVATE)` on the test class
   - Field order: `@Captor` → `@Mock` → `@Spy` → `@InjectMocks` → auxiliary fields
   - `initValues()` + `prepareMocks()` called from `@BeforeEach setUp()`
   - `doReturn(...).when(...)` — never `when(...).thenReturn(...)`
   - `assertDoesNotThrow(() -> ...)` for every happy-path call
   - `assertThatThrownBy(...)` for exception scenarios
   - AssertJ fluent chaining with `//` continuation
   - `testShould<Action>()` method name convention
   - `"Given ..., when ..., then ..."` `@DisplayName` format
   - Parameterized tests for Factories and Mappers

## Target stack

Java 17+ · JUnit 5 · Mockito · AssertJ · Lombok

## Usage

```
/java-junit-mockito-tests CreateOrderUseCaseImpl
/java-junit-mockito-tests src/main/java/br/com/example/OrderGateway.java
```

## Notes

- `@Mock` for dependencies with other dependencies or Spring Data repositories
- `@Spy` for mappers and utilities instantiable with `new`
- Helper classes use `@UtilityClass` with fixed UUIDs — never `UUID.randomUUID()`
- Builder classes use `@NoArgsConstructor(access = PRIVATE)` with a static factory method
