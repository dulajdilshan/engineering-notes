

### Precision in Testing: Leveraging Slice Tests for Efficient Coverage in Spring Boot

In many real-world Spring Boot projects, especially those handled by **small teams (2–3 developers)**, the pressure to deliver quickly is constant. Clients often begin with a request for a *minimal viable product (MVP)* or a small, functional application that can be deployed fast. In such scenarios, teams prioritize building features rapidly—sometimes skipping or minimizing tests just to get the system running.

While this approach helps in meeting early delivery goals, it can create challenges later when the system evolves. Adding new features or debugging regressions without solid test coverage often slows down development over time. This is where **slice testing** becomes a practical and effective strategy.

<img width="460" height="376" alt="image" src="https://github.com/user-attachments/assets/51687c9f-b1c7-4ec9-9b7a-28bbf01400e5" />

<img width="460" height="376" alt="image" src="https://github.com/user-attachments/assets/731cae87-d5e0-4fd5-a5bd-d5e370003fcb" />

---

#### ⚙️ Slice Testing for Small Teams and Fast Delivery

Slice testing enables smaller teams to achieve **high test coverage with minimal effort and faster execution**. Instead of writing separate unit tests for each layer—controller, service, and repository—we can focus on **controller and service layers together**, while **mocking the repository or database layer**. This approach provides several benefits:

* **Speed of development**: Fewer tests are needed to validate the same logic paths, allowing teams to iterate quickly.
* **Focused coverage**: Both controller and service logic are validated in one test, ensuring request validation, response correctness, and business rules are all verified.
* **Ease of maintenance**: With fewer test classes and less duplication, maintaining the test suite is simpler and faster.
* **Reliability during refactors**: Since slice tests validate multiple layers in one pass, they catch integration-level issues early—without requiring full-blown integration tests.

This makes slice testing ideal for small, fast-moving teams where **time-to-market** and **reliability** must coexist.

---

#### 🎯 The Essence of Slice Testing

Slice testing focuses on validating a vertical slice of the application—typically the **controller** and **service** layers—while **mocking dependencies** such as the repository or external APIs. This enables verification of cross-layer logic without the overhead of loading the full application context or connecting to a real database.

Spring Boot provides several specialized annotations that simplify this approach:

* `@WebMvcTest` for controller-focused slices
* `@DataJpaTest` for repository validation
* `@SpringBootTest` for full integration tests

By combining `@WebMvcTest` with `@MockBean` to mock dependencies, developers can simulate realistic scenarios and validate the most critical paths of their API quickly and reliably.

---

#### 🧠 When to Use Slice Tests vs Unit Tests

* Use **unit tests** for **pure logic or utility classes** that have no Spring dependencies.
* Use **slice tests** for **controller + service interactions**, mocking the repository or external systems.
* Use **integration tests** sparingly, for **end-to-end scenarios** that are critical to business flow.

This hybrid testing approach ensures high confidence in behavior with minimal duplication—ideal for small teams balancing agility and quality.

---

#### 🧩 Example: Controller-Service Slice Test

Below is a simplified slice test extracted and adapted from a real-world Spring Boot project. It demonstrates how to test both the **controller** and **service** layers together, while mocking the **repository** layer for controlled, deterministic behavior.

```java
@SpringBootTest
@AutoConfigureMockMvc
class StudentProfileControllerSliceTest extends AbstractMockMvcTest {

    private Authentication auth;
    private Student student;

    @BeforeEach
    @Override
    public void setUp() {
        auth = EntityTestUtil.testAuth(1, Enums.ERole.ROLE_TEACHER);
        student = EntityTestUtil.testStudent(10);
        super.setUp();
    }

    @Test
    void getStudentProfile_ok() throws Exception {
        // Arrange
        when(super.studentRepository.findById(10))
                .thenReturn(Optional.of(student));

        // Act & Assert
        mockMvc.perform(get("/v1/teacher/profile/10")
                        .param("role", "ROLE_STUDENT")
                        .with(authentication(auth)))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.id").value(10))
                .andExpect(jsonPath("$.user.username").value(student.getUser().getUsername()));
    }
}
```

**Key Points:**

* `@SpringBootTest` combined with `@AutoConfigureMockMvc` loads only the necessary components for the test.
* The **repository layer** (`studentRepository`) is mocked, preventing database interactions.
* The test validates both the **controller response** and the **business logic flow** in the **service layer**.
* Common setup logic is inherited from `AbstractMockMvcTest`, ensuring clean and reusable configurations.

This simple example demonstrates how a slice test can verify the controller and service layers in one go, mocking out repository calls for predictable results.

---

#### 🚀 Takeaway

For smaller teams focused on fast delivery, **slice testing** offers the perfect balance between speed, reliability, and maintainability. By mocking the repository or database layer and testing controller-service interactions together, we can:

* Deliver new features rapidly,
* Ensure robust API-level validation, and
* Maintain confidence in business logic without writing redundant tests.

This approach embodies a pragmatic engineering mindset—delivering **clean, reliable, and scalable code** while keeping pace with the demands of modern software delivery.
