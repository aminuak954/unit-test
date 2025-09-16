
# Guidelines for Writing Unit Tests in Spring Boot with JUnit 5 and Mockito

This guideline outlines best practices for writing unit tests in a Spring Boot application using JUnit 5 and Mockito. It covers setting up tests, mocking dependencies, testing Spring components (e.g., services, controllers), and handling common Spring Boot scenarios.

## 1. Introduction

Unit tests in Spring Boot focus on testing individual components (e.g., services, controllers) in isolation, mocking dependencies like repositories, external services, or other beans. JUnit 5 provides a modern testing framework, and Mockito is used to mock dependencies. For Spring-specific features, Spring Boot's testing support (via `@SpringBootTest`, `@WebMvcTest`, etc.) is often combined with Mockito for fine-grained control.

### Key Principles
- **Isolation**: Test a single component, mocking all external dependencies (e.g., repositories, REST clients).
- **Speed**: Unit tests should be fast, avoiding Spring context loading unless necessary.
- **Clarity**: Tests should be readable, acting as documentation for the component's behavior.
- **Repeatability**: Tests should produce consistent results, unaffected by external systems (e.g., databases).
- **Independence**: Each test should set up its own data and not rely on other tests.

## 2. Setting Up Your Spring Boot Project

Add the necessary dependencies for JUnit 5, Mockito, and Spring Boot testing in your `pom.xml` (Maven) or `build.gradle` (Gradle).

### Maven Dependencies
```xml
<dependencies>
    <!-- Spring Boot Test Starter (includes JUnit 5) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <version>${spring-boot.version}</version> <!-- e.g., 3.2.5 -->
        <scope>test</scope>
    </dependency>
    <!-- Mockito Core (optional, included in spring-boot-starter-test) -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.11.0</version>
        <scope>test</scope>
    </dependency>
    <!-- Mockito JUnit 5 Extension -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>5.11.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle Dependencies
```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.mockito:mockito-core:5.11.0'
    testImplementation 'org.mockito:mockito-junit-jupiter:5.11.0'
}
```

The `spring-boot-starter-test` includes JUnit 5, Mockito, AssertJ, and other testing utilities. Ensure your Spring Boot version (e.g., 3.2.5 as of September 2025) is consistent across dependencies.

## 3. Organizing Test Code

### Naming Conventions
- **Test Classes**: Name as `ClassUnderTestTest`, e.g., `UserServiceTest`, `UserControllerTest`.
- **Test Methods**: Use descriptive names like `shouldReturnUserWhenIdIsValid()` or `shouldThrowExceptionWhenInputIsNull()`.
- **Packages**: Mirror production code, e.g., `com.example.service` for production code has `com.example.service` for tests.

### Test Structure
Follow the **Arrange-Act-Assert (AAA)** pattern:
- **Arrange**: Set up test data, mocks, and initial state.
- **Act**: Call the method under test.
- **Assert**: Verify the result or mock interactions.

### Directory Structure
Place tests in `src/test/java`, mirroring `src/main/java`. Resources (e.g., test configuration files) go in `src/test/resources`.

## 4. Writing Unit Tests for Spring Boot Components

Spring Boot applications typically involve testing **controllers**, **services**, and **repositories**. For unit tests, focus on isolating the component using Mockito and avoid loading the full Spring context unless necessary.

### 4.1 Testing a Service
Consider a `UserService` that depends on a `UserRepository`:
```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User getUserById(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("User not found"));
    }

    public User saveUser(User user) {
        if (user.getName() == null) {
            throw new IllegalArgumentException("Name cannot be null");
        }
        return userRepository.save(user);
    }
}
```

Test class using JUnit 5 and Mockito:
```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.util.Optional;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnUserWhenIdExists() {
        // Arrange
        Long id = 1L;
        User mockUser = new User(id, "John Doe");
        when(userRepository.findById(id)).thenReturn(Optional.of(mockUser));

        // Act
        User result = userService.getUserById(id);

        // Assert
        assertEquals(mockUser, result);
        verify(userRepository).findById(id);
    }

    @Test
    void shouldThrowExceptionWhenUserNotFound() {
        // Arrange
        Long id = 999L;
        when(userRepository.findById(id)).thenReturn(Optional.empty());

        // Act & Assert
        RuntimeException exception = assertThrows(RuntimeException.class, () -> userService.getUserById(id));
        assertEquals("User not found", exception.getMessage());
        verify(userRepository).findById(id);
    }

    @Test
    void shouldSaveUserWhenValid() {
        // Arrange
        User user = new User(1L, "John Doe");
        when(userRepository.save(any(User.class))).thenReturn(user);

        // Act
        User result = userService.saveUser(user);

        // Assert
        assertEquals(user, result);
        verify(userRepository).save(user);
    }

    @Test
    void shouldThrowExceptionWhenNameIsNull() {
        // Arrange
        User user = new User(1L, null);

        // Act & Assert
        IllegalArgumentException exception = assertThrows(IllegalArgumentException.class, () -> userService.saveUser(user));
        assertEquals("Name cannot be null", exception.getMessage());
        verify(userRepository, never()).save(any());
    }
}
```

**Key Points**:
- Use `@ExtendWith(MockitoExtension.class)` to enable Mockito annotations.
- `@Mock` creates a mock for `UserRepository`.
- `@InjectMocks` injects the mock into `UserService` via constructor injection.
- Use `when().thenReturn()` for stubbing and `verify()` for interaction checks.
- Test both happy paths and edge cases (e.g., null inputs, missing entities).

### 4.2 Testing a REST Controller
For controllers, use `@WebMvcTest` to load only the web layer, mocking service dependencies.

Example `UserController`:
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.getUserById(id));
    }

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        return ResponseEntity.status(HttpStatus.CREATED).body(userService.saveUser(user));
    }
}
```

Test class:
```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUserWhenIdExists() throws Exception {
        // Arrange
        Long id = 1L;
        User user = new User(id, "John Doe");
        when(userService.getUserById(id)).thenReturn(user);

        // Act & Assert
        mockMvc.perform(get("/api/users/{id}", id))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(id))
                .andExpect(jsonPath("$.name").value("John Doe"));
        verify(userService).getUserById(id);
    }

    @Test
    void shouldReturnCreatedUser() throws Exception {
        // Arrange
        User user = new User(1L, "John Doe");
        when(userService.saveUser(any(User.class))).thenReturn(user);

        // Act & Assert
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"id\":1,\"name\":\"John Doe\"}"))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("John Doe"));
        verify(userService).saveUser(any(User.class));
    }
}
```

**Key Points**:
- `@WebMvcTest(UserController.class)` loads only the web layer for `UserController`.
- `@MockBean` mocks the `UserService` bean, replacing the real one in the Spring context.
- Use `MockMvc` to simulate HTTP requests and verify responses (status, JSON content).
- Use `jsonPath` to assert JSON response fields.
- Test HTTP status codes, response bodies, and service interactions.

### 4.3 Testing Repositories
For unit tests, mock repositories to avoid hitting the database. However, if you need to test repository logic (e.g., custom queries), use `@DataJpaTest`.

Example `UserRepository`:
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByName(String name);
}
```

Test with `@DataJpaTest` (integration-style, not pure unit test):
```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import static org.junit.jupiter.api.Assertions.*;

@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindUserByName() {
        // Arrange
        User user = new User(1L, "John Doe");
        userRepository.save(user);

        // Act
        Optional<User> result = userRepository.findByName("John Doe");

        // Assert
        assertTrue(result.isPresent());
        assertEquals("John Doe", result.get().getName());
    }
}
```

**Key Points**:
- `@DataJpaTest` loads only the JPA layer with an in-memory database (e.g., H2).
- Use for testing custom repository methods or JPA queries.
- For pure unit tests, mock the repository in service tests instead of using `@DataJpaTest`.

## 5. Advanced Testing Features

### 5.1 Parameterized Tests
Use `@ParameterizedTest` for data-driven tests.
```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @ParameterizedTest
    @CsvSource({"1, John", "2, Jane"})
    void shouldReturnUserForValidId(Long id, String name) {
        // Arrange
        User mockUser = new User(id, name);
        when(userRepository.findById(id)).thenReturn(Optional.of(mockUser));

        // Act
        User result = userService.getUserById(id);

        // Assert
        assertEquals(name, result.getName());
        verify(userRepository).findById(id);
    }
}
```

### 5.2 Testing Asynchronous Code
For async methods (e.g., using `@Async`), use `CompletableFuture` or await mechanisms.
```java
@Service
public class AsyncService {
    @Async
    public CompletableFuture<String> processAsync() {
        return CompletableFuture.completedFuture("Done");
    }
}

@ExtendWith(MockitoExtension.class)
class AsyncServiceTest {

    @InjectMocks
    private AsyncService asyncService;

    @Test
    void shouldCompleteAsyncTask() throws Exception {
        // Act
        CompletableFuture<String> result = asyncService.processAsync();

        // Assert
        assertEquals("Done", result.get());
    }
}
```

**Note**: Ensure `@EnableAsync` is configured in your application for async tests.

### 5.3 Mocking Static Methods
Mock static methods (e.g., utility classes) with `mockStatic`:
```java
@Test
void shouldHandleStaticUtility() {
    try (MockedStatic<Utility> mocked = mockStatic(Utility.class)) {
        // Arrange
        mocked.when(Utility::getCurrentTime).thenReturn("2025-09-16");
        
        // Act
        String result = Utility.getCurrentTime();
        
        // Assert
        assertEquals("2025-09-16", result);
    }
}
```

## 6. Best Practices

### General Practices
- **Focus on Unit Tests**: Use Mockito to mock dependencies and avoid loading the full Spring context with `@SpringBootTest` for unit tests.
- **One Behavior Per Test**: Test a single scenario per test method; use `assertAll()` for multiple related assertions.
- **Test Edge Cases**: Cover null inputs, empty results, exceptions, and boundary conditions.
- **Descriptive Names**: Use method names like `shouldXWhenY` to describe intent.
- **Keep Tests Fast**: Avoid database or network calls in unit tests by mocking dependencies.
- **Avoid Over-Mocking**: Mock only external dependencies (e.g., repositories, external services), not the class under test.

### Spring-Specific Practices
- **Use `@WebMvcTest` for Controllers**: Test only the web layer, mocking services with `@MockBean`.
- **Use `@DataJpaTest` Sparingly**: Reserve for testing JPA repositories; mock repositories in service unit tests.
- **Mock External Services**: Mock REST clients (e.g., `RestTemplate`, `WebClient`) to avoid external calls.
- **Test Configuration**: Place test-specific properties in `src/test/resources/application.properties` or use `@TestPropertySource`.
- **Verify HTTP Responses**: For controllers, check status codes, headers, and JSON content with `MockMvc`.

### Mockito-Specific Practices
- **Use Argument Matchers**: Use `any()`, `eq()`, or `argThat()` for flexible stubbing: `when(repository.save(any(User.class))).thenReturn(user)`.
- **Verify Interactions**: Use `verify(mock).method()` for critical interactions, but prefer state verification (checking results) when possible.
- **Use `@MockBean` for Spring Context**: In `@WebMvcTest` or `@SpringBootTest`, use `@MockBean` to replace Spring beans with mocks.
- **Avoid Mocking Value Objects**: Don’t mock simple data classes (e.g., DTOs).
- **Use ArgumentCaptor**: Capture arguments for detailed verification:
  ```java
  ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
  verify(userRepository).save(captor.capture());
  assertEquals("John Doe", captor.getValue().getName());
  ```

### Common Pitfalls to Avoid
- **Loading Full Context**: Avoid `@SpringBootTest` for unit tests; use `@WebMvcTest` or `@ExtendWith(MockitoExtension.class)`.
- **Flaky Tests**: Ensure tests don’t rely on external systems, time, or random data.
- **Over-Verifying Mocks**: Don’t verify every interaction; focus on key behaviors.
- **Ignoring Test Failures**: Run tests frequently (`mvn test` or `./gradlew test`) and fix issues promptly.
- **Not Cleaning Up**: Use `@AfterEach` for cleanup if shared state is unavoidable (rare in unit tests).

## 7. Example: Testing a Complete Flow

### Scenario
A `UserService` calls `UserRepository` and an external `EmailService`.

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }

    public User createUser(User user) {
        User savedUser = userRepository.save(user);
        emailService.sendWelcomeEmail(user.getName());
        return savedUser;
    }
}
```

Test:
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldCreateUserAndSendEmail() {
        // Arrange
        User user = new User(1L, "John Doe");
        when(userRepository.save(any(User.class))).thenReturn(user);
        doNothing().when(emailService).sendWelcomeEmail(anyString());

        // Act
        User result = userService.createUser(user);

        // Assert
        assertEquals(user, result);
        verify(userRepository).save(user);
        verify(emailService).sendWelcomeEmail("John Doe");
    }
}
```

**Key Points**:
- Mock both `UserRepository` and `EmailService`.
- Use `doNothing()` for void methods like `sendWelcomeEmail`.
- Verify both the saved user and the email interaction.

## 8. Additional Tips for Spring Boot

- **Test Configuration**: Use `@TestConfiguration` to define test-specific beans if needed.
- **Mocking RestTemplate/WebClient**: Mock HTTP clients to simulate external API calls:
  ```java
  @MockBean
  private RestTemplate restTemplate;
  
  @Test
  void shouldCallExternalApi() {
      when(restTemplate.getForObject(anyString(), eq(String.class))).thenReturn("Response");
      // Test logic
  }
  ```
- **Test Profiles**: Use `@ActiveProfiles("test")` to load test-specific configurations.
- **Coverage Reports**: Use tools like JaCoCo (`mvn jacoco:report`) to measure test coverage, aiming for 80-90% for critical components.

## 9. Resources
- **JUnit 5 Documentation**: https://junit.org/junit5/docs/current/user-guide/
- **Mockito Documentation**: https://site.mockito.org/
- **Spring Boot Testing**: https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing
