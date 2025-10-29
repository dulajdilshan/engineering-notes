# **Comprehensive Automated Testing Strategies for Spring Boot APIs: A Production-Grade Approach**

## **1\. Why Testing Backend Applications is Mission-Critical**

In modern distributed systems, backend APIs serve as the backbone of enterprise applications. A single undetected bug can cascade into service degradation, data corruption, or security vulnerabilities.

While manual testing can verify functionality, it is inherently slow, error-prone, and unscalable. An automated testing strategy is not just about *verification*; it is a business imperative for enabling teams to move fast, with confidence.

**Key Imperatives for an Automated API Testing Strategy:**

* **Confidence and Reliability**: Automated tests provide confidence that the code does what it should, from business logic to security policies. This ensures reliability under scale, maintaining SLA commitments under peak traffic and gracefully degrading during resource constraints.  
* **Velocity and Fast Feedback**: By automating the slow, boring, and error-prone process of manual testing, we create a CI/CD pipeline that provides fast, accurate, and predictable feedback. This allows teams to iterate rapidly and deploy changes with high confidence, rather than fear.  
* **Maintainability and Contract Compliance**: A well-written test suite is a form of executable documentation that makes maintenance easier. For APIs, this extends to *contract compliance*. Comprehensive testing validates backward compatibility, preventing breaking changes that can ripple across a microservices ecosystem.  
* **Security Assurance**: APIs expose attack surfaces that require rigorous validation. Testing identifies vulnerabilities in authentication, authorization, and input validation before they can be exploited.


## **2\. The Testing Model: From Pyramid to Honeycomb**

The classic **Testing Pyramid** advocates for a large base of fast unit tests, a smaller layer of integration tests, and a small peak of end-to-end tests.

However, for Spring Boot microservices—which are often integration-heavy (connecting controllers, services, repositories, and databases) and lighter on complex, isolated logic—this model can be insufficient.

We advocate for a **"Testing Honeycomb"** model, which values integration tests more highly. This model acknowledges that the *highest risk* in a microservice often lies in the *interaction* of its components. Another similar and popular concept is the **"Testing Trophy"** model, which also emphasizes integration tests as the most valuable and largest category of tests, supported by a solid foundation of unit tests and a few end-to-end or static analysis checks.

Our strategy, therefore, focuses on multiple layers of integration testing for maximum confidence.

### **2.1. Unit Testing (The Foundation)**

Unit tests validate individual components (e.g., a single service method) in isolation. Dependencies are mocked using tools like Mockito. These tests are fast and form the foundation of our strategy.

**Example: Testing LMS Course Service Logic**

```java
@ExtendWith(MockitoExtension.class)  
class CourseServiceTest {  
      
    @Mock  
    private CourseRepository courseRepository;  
      
    @Mock  
    private EnrollmentRepository enrollmentRepository;  
      
    @InjectMocks  
    private CourseService courseService;  
      
    @Test  
    @DisplayName("Should successfully enroll student when course has available capacity")  
    void testEnrollStudent\_Success() {  
        // Arrange  
        Long courseId \= 1L;  
        Long studentId \= 100L;  
        Course course \= Course.builder()  
            .id(courseId)  
            .title("Advanced System Design")  
            .maxCapacity(30)  
            .currentEnrollment(25)  
            .build();  
              
        when(courseRepository.findById(courseId))  
            .thenReturn(Optional.of(course));  
        when(enrollmentRepository.existsByCourseIdAndStudentId(courseId, studentId))  
            .thenReturn(false);  
          
        // Act  
        EnrollmentResult result \= courseService.enrollStudent(courseId, studentId);  
          
        // Assert  
        assertThat(result.isSuccess()).isTrue();  
        assertThat(course.getCurrentEnrollment()).isEqualTo(26);  
        verify(courseRepository).save(course);  
        verify(enrollmentRepository).save(any(Enrollment.class));  
    }  
      
    @Test  
    @DisplayName("Should reject enrollment when course is at full capacity")  
    void testEnrollStudent\_CourseAtCapacity() {  
        // Arrange  
        Course fullCourse \= Course.builder()  
            .id(1L)  
            .maxCapacity(30)  
            .currentEnrollment(30)  
            .build();  
              
        when(courseRepository.findById(1L))  
            .thenReturn(Optional.of(fullCourse));  
          
        // Act & Assert  
        assertThrows(CourseFullException.class,   
            () \-\> courseService.enrollStudent(1L, 100L));  
        verify(enrollmentRepository, never()).save(any());  
    }  
      
    @Test  
    @DisplayName("Should prevent duplicate enrollment for same student")  
    void testEnrollStudent\_DuplicateEnrollment() {  
        // Arrange  
        when(courseRepository.findById(1L))  
            .thenReturn(Optional.of(createCourse())); // Assumes a helper method  
        when(enrollmentRepository.existsByCourseIdAndStudentId(1L, 100L))  
            .thenReturn(true);  
          
        // Act & Assert  
        assertThrows(DuplicateEnrollmentException.class,  
            () \-\> courseService.enrollStudent(1L, 100L));  
    }  
}
```

### **2.2. Slice Integration Testing (The Middle Layer)**

This layer tests a specific "slice" of the application context, providing a balance between unit test speed and full integration test realism. We use Spring Boot's test slices.

**Example 1: Testing the Web Layer (@WebMvcTest)**

This test loads only the web layer (controllers, JSON serialization, security) and mocks the service layer. It's ideal for validating API contracts, input validation (@Valid), and controller-level security.

```java
@WebMvcTest(CourseController.class)  
// Secure this endpoint (if using Spring Security)  
@Import(TestSecurityConfig.class)   
class CourseControllerSliceTest {

    @Autowired  
    private MockMvc mockMvc;

    @MockBean  
    private CourseService courseService;

    @Autowired  
    private ObjectMapper objectMapper;

    @Test  
    @DisplayName("POST /api/v1/courses should return 400 Bad Request for invalid input")  
    void testCreateCourse\_InvalidInput() throws Exception {  
        // Arrange  
        CourseCreateRequest invalidRequest \= new CourseCreateRequest();  
        invalidRequest.setTitle(""); // Title is blank, violating @NotBlank  
          
        // Act & Assert  
        mockMvc.perform(post("/api/v1/courses")  
                .with(SecurityMockMvcRequestPostProcessors.jwt()) // Mock auth  
                .contentType(MediaType.APPLICATION\_JSON)  
                .content(objectMapper.writeValueAsString(invalidRequest)))  
            .andExpect(status().isBadRequest())  
            .andExpect(jsonPath("$.errors.title").value("Title must not be blank"));  
              
        verify(courseService, never()).createCourse(any());  
    }  
}
```

**Example 2: Testing the Persistence Layer (@DataJpaTest)**

This test loads only the JPA context (repositories, entity mappings). It's perfect for validating custom queries, indexes, and entity relationships. It can run against an embedded DB or, preferably, Testcontainers.

```java
@DataJpaTest  
// Use Testcontainers for a real database  
@AutoConfigureTestDatabase(replace \= AutoConfigureTestDatabase.Replace.NONE)   
@Testcontainers  
class CourseRepositorySliceTest {

    @Container  
    static PostgreSQLContainer\<?\> postgres \= new PostgreSQLContainer\<\>("postgres:15-alpine");

    @DynamicPropertySource  
    static void configureProperties(DynamicPropertyRegistry registry) {  
        registry.add("spring.datasource.url", postgres::getJdbcUrl);  
        registry.add("spring.datasource.username", postgres::getUsername);  
        registry.add("spring.datasource.password", postgres::getPassword);  
    }

    @Autowired  
    private TestEntityManager entityManager;

    @Autowired  
    private CourseRepository courseRepository;

    @Test  
    @DisplayName("Should find all advanced courses with enrollment below capacity")  
    void testFindAvailableAdvancedCourses() {  
        // Arrange  
        entityManager.persist(new Course("Advanced Java", CourseLevel.ADVANCED, 50, 49));  
        entityManager.persist(new Course("Intro to Java", CourseLevel.BEGINNER, 50, 30));  
        entityManager.persist(new Course("Advanced SQL", CourseLevel.ADVANCED, 50, 50)); // Full  
          
        // Act  
        List\<Course\> results \= courseRepository.findAvailableAdvancedCourses();  
          
        // Assert  
        assertThat(results).hasSize(1);  
        assertThat(results.get(0).getTitle()).isEqualTo("Advanced Java");  
    }  
}
```

### **2.3. Full-Context Integration Testing (The Core)**

This is the most comprehensive test, loading the *entire* Spring Boot application context (@SpringBootTest). It validates the full flow, from an HTTP request through the service and persistence layers to a real database (via Testcontainers).

**Example: Testing Endpoints with TestContainers**

```java
@SpringBootTest(webEnvironment \= SpringBootTest.WebEnvironment.RANDOM\_PORT)  
@Testcontainers  
@ActiveProfiles("test")  
class CourseControllerIntegrationTest {  
      
    @Container  
    static PostgreSQLContainer\<?\> postgres \= new PostgreSQLContainer\<\>("postgres:15-alpine")  
        .withDatabaseName("lms\_test");  
      
    @Autowired  
    private TestRestTemplate restTemplate;  
      
    @Autowired  
    private CourseRepository courseRepository;  
      
    @DynamicPropertySource  
    static void configureProperties(DynamicPropertyRegistry registry) {  
        registry.add("spring.datasource.url", postgres::getJdbcUrl);  
        registry.add("spring.datasource.username", postgres::getUsername);  
        registry.add("spring.datasource.password", postgres::getPassword);  
    }  
      
    @BeforeEach  
    void setUp() {  
        courseRepository.deleteAll();  
    }  
      
    @Test  
    @DisplayName("Should return paginated course list with correct metadata")  
    void testGetCourses\_Pagination() {  
        // Arrange  
        createTestCourses(25); // Helper to create 25 courses  
          
        // Act  
        ResponseEntity\<PagedResponse\<CourseDTO\>\> response \= restTemplate.exchange(  
            "/api/v1/courses?page=0\&size=10\&sort=title,asc",  
            HttpMethod.GET,  
            null,  
            new ParameterizedTypeReference\<\>() {}  
        );  
          
        // Assert  
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);  
        PagedResponse\<CourseDTO\> body \= response.getBody();  
        assertThat(body.getContent()).hasSize(10);  
        assertThat(body.getTotalElements()).isEqualTo(25);  
        assertThat(body.getTotalPages()).isEqualTo(3);  
    }  
      
    @Test  
    @DisplayName("Should handle concurrent enrollment requests with optimistic locking")  
    void testConcurrentEnrollment\_OptimisticLocking() throws InterruptedException {  
        // Arrange  
        Course course \= createCourseWithCapacity(2);  
        int numberOfThreads \= 5;  
        CountDownLatch latch \= new CountDownLatch(numberOfThreads);  
        List\<CompletableFuture\<ResponseEntity\<EnrollmentDTO\>\>\> futures \= new ArrayList\<\>();  
          
        // Act \- Simulate 5 concurrent enrollment attempts for 2 spots  
        for (int i \= 0; i \< numberOfThreads; i++) {  
            final int studentId \= 100 \+ i;  
            CompletableFuture\<ResponseEntity\<EnrollmentDTO\>\> future \=   
                CompletableFuture.supplyAsync(() \-\> {  
                    latch.countDown();  
                    try {  
                        latch.await();  
                    } catch (InterruptedException e) {  
                        Thread.currentThread().interrupt();  
                    }  
                    return enrollStudent(course.getId(), studentId);  
                });  
            futures.add(future);  
        }  
          
        List\<ResponseEntity\<EnrollmentDTO\>\> responses \= futures.stream()  
            .map(CompletableFuture::join)  
            .collect(Collectors.toList());  
          
        // Assert  
        long successfulEnrollments \= responses.stream()  
            .filter(r \-\> r.getStatusCode() \== HttpStatus.CREATED)  
            .count();  
        long rejectedEnrollments \= responses.stream()  
            // 409 Conflict is expected from optimistic lock failures or full capacity  
            .filter(r \-\> r.getStatusCode() \== HttpStatus.CONFLICT)   
            .count();  
              
        assertThat(successfulEnrollments).isEqualTo(2);  
        assertThat(rejectedEnrollments).isEqualTo(3);  
          
        Course updatedCourse \= courseRepository.findById(course.getId()).get();  
        assertThat(updatedCourse.getCurrentEnrollment()).isEqualTo(2);  
    }  
}
```

## **3\. Contract Testing**

In a microservices architecture, contract testing ensures a *consumer* (another service) and a *provider* (this API) can communicate. It prevents breaking changes.

**Example: Spring Cloud Contract for LMS Grade Service**

The *provider* (our service) defines a contract. This test auto-generates from the contract, starts our app, and verifies the contract is met.

```java
@SpringBootTest(webEnvironment \= SpringBootTest.WebEnvironment.MOCK)  
@AutoConfigureMessageVerifier  
class GradeServiceContractTest {  
      
    @Autowired  
    private GradeService gradeService;  
      
    @MockBean  
    private AssignmentRepository assignmentRepository;  
      
    @BeforeEach  
    void setUp() {  
        RestAssuredMockMvc.standaloneSetup(new GradeController(gradeService));  
          
        Assignment assignment \= Assignment.builder()  
            .id(1L)  
            .title("Microservices Design Assignment")  
            .maxScore(100)  
            .build();  
              
        when(assignmentRepository.findById(1L))  
            .thenReturn(Optional.of(assignment));  
          
        // Mock service logic to return a specific grade  
        when(gradeService.getGradeForStudent(1L, 100L))  
            .thenReturn(createGradeDto());   
    }  
}
```

**Contract Definition (Groovy DSL or YAML)**

This contract defines the expected request and response, which is shared with consumer teams.
```yml
description: Get student grades for an assignment  
request:  
  method: GET  
  url: /api/v1/assignments/1/grades/student/100  
response:  
  status: 200  
  headers:  
    Content-Type: application/json  
  body:  
    assignmentId: 1  
    studentId: 100  
    score: 85  
    maxScore: 100  
    submittedAt: "2025-01-15T10:30:00Z"  
  matchers:  
    body:  
      \- path: $.score  
        type: by\_regex  
        value: "\[0-9\]+"  
      \- path: $.submittedAt  
        type: by\_regex  
        value: "\\\\d{4}-\\\\d{2}-\\\\d{2}T\\\\d{2}:\\\\d{2}:\\\\d{2}Z"
```

## **4\. Performance Testing**

Performance tests validate API behavior (latency, throughput) under load. We use Gatling to script user scenarios and identify bottlenecks.

**Example: Gatling Simulation for LMS Search API**

```java
public class CourseSearchLoadTest extends Simulation {  
      
    private HttpProtocolBuilder httpProtocol \= http  
        .baseUrl("http://localhost:8080")  
        .acceptHeader("application/json")  
        .header("Authorization", "Bearer ${jwt\_token}");  
      
    private ScenarioBuilder searchScenario \= scenario("Course Search Load Test")  
        .exec(  
            http("Search by keyword")  
                .get("/api/v1/courses/search?q=microservices")  
                .check(status().is(200))  
                .check(jsonPath("$.content").exists())  
                .check(responseTimeInMillis().lt(500)) // p95 \< 500ms  
        );  
      
    {  
        setUp(  
            searchScenario.injectOpen(  
                rampUsersPerSec(10).to(100).during(Duration.ofMinutes(2)),  
                constantUsersPerSec(100).during(Duration.ofMinutes(5))  
            )  
        ).protocols(httpProtocol)  
         .assertions(  
             global().responseTime().percentile3().lt(1000), // p99 \< 1s  
             global().successfulRequests().percent().gt(99.5)  
         );  
    }  
}
```

## **5\. Security Testing**

Security tests are integrated into our test suite to validate authentication, authorization, and input validation.

**Example: Security Test Suite for LMS API**

```java
@SpringBootTest(webEnvironment \= SpringBootTest.WebEnvironment.RANDOM\_PORT)  
@AutoConfigureMockMvc  
class SecurityIntegrationTest {  
      
    @Autowired  
    private MockMvc mockMvc;

    @Autowired  
    private CourseRepository courseRepository;  
      
    @Test  
    @DisplayName("Should reject requests without authentication token")  
    void testUnauthenticatedAccess\_Rejected() throws Exception {  
        mockMvc.perform(get("/api/v1/courses"))  
            .andExpect(status().isUnauthorized());  
    }

    @Test  
    @DisplayName("Should enforce authorization for course management")  
    @WithMockUser(username \= "student", roles \= {"STUDENT"})  
    void testDeleteCourse\_AsStudent\_Forbidden() throws Exception {  
        // Arrange  
        Course course \= courseRepository.save(new Course("Test Course"));  
          
        // Act & Assert  
        mockMvc.perform(delete("/api/v1/courses/" \+ course.getId()))  
            .andExpect(status().isForbidden());  
    }

    @Test  
    @DisplayName("Should sanitize SQL injection attempts in search queries")  
    @WithMockUser(roles \= {"STUDENT"})  
    void testSqlInjection\_Sanitized() throws Exception {  
        String maliciousQuery \= "'; DROP TABLE courses; \--";  
          
        mockMvc.perform(get("/api/v1/courses/search")  
                .param("q", maliciousQuery))  
            .andExpect(status().isOk())  
            .andExpect(jsonPath("$.content").isArray());  
          
        // Verify table still exists (the real check)  
        assertThat(courseRepository.count()).isGreaterThanOrEqualTo(0);  
    }  
}
```

## **6\. Advanced Testing Patterns**

### **6.1. Test Data Builders**

Implement builder patterns for complex test data creation, improving test readability and maintenance.

```java
public class CourseTestDataBuilder {  
      
    private String title \= "Default Course";  
    private Integer maxCapacity \= 30;  
    private Integer currentEnrollment \= 0;  
      
    public static CourseTestDataBuilder aCourse() {  
        return new CourseTestDataBuilder();  
    }  
      
    public CourseTestDataBuilder withTitle(String title) {  
        this.title \= title;  
        return this;  
    }  
      
    public CourseTestDataBuilder atFullCapacity() {  
        this.currentEnrollment \= this.maxCapacity;  
        return this;  
    }  
      
    public Course build() {  
        return Course.builder()  
            .title(title)  
            .maxCapacity(maxCapacity)  
            .currentEnrollment(currentEnrollment)  
            .build();  
    }  
}
```

```java
// Usage in test:  
Course fullCourse \= CourseTestDataBuilder.aCourse()  
    .withTitle("Advanced Concurrency")  
    .atFullCapacity()  
    .build();
```

### **6.2. Test Fixtures with Database Migrations**

Use Flyway or Liquibase for consistent test database state. Test data can be loaded via migration scripts placed in src/test/resources/db/testdata.

```java
@TestConfiguration  
public class TestDatabaseConfig {  
      
    @Bean  
    public Flyway flyway(DataSource dataSource) {  
        Flyway flyway \= Flyway.configure()  
            .dataSource(dataSource)  
            // Add test-specific data SQL scripts  
            .locations("classpath:db/migration", "classpath:db/testdata")   
            .cleanDisabled(false)  
            .load();  
          
        flyway.clean();  
        flyway.migrate();  
        return flyway;  
    }  
}
```

### **6.3. Faking External Dependencies with WireMock**

When our service calls an external HTTP API (e.g., a Notification Service), we must not mock our RestTemplate or WebClient. Instead, we test the real HTTP call against a fake server, **WireMock**.

**Example: Testing Enrollment Notification**

```java
@SpringBootTest(webEnvironment \= SpringBootTest.WebEnvironment.RANDOM\_PORT)  
@ActiveProfiles("test")  
// Register WireMock as a test extension  
@ExtendWith(WireMockExtension.class)  
class EnrollmentNotificationTest {

    @Autowired  
    private TestRestTemplate restTemplate;

    // Inject the WireMock server; assumes it runs on port 9090  
    static WireMockServer wireMockServer \= new WireMockServer(9090);

    @BeforeAll  
    static void startWireMock() {  
        wireMockServer.start();  
    }

    @AfterAll  
    static void stopWireMock() {  
        wireMockServer.stop();  
    }

    @BeforeEach  
    void resetWireMock() {  
        wireMockServer.resetAll();  
    }

    @DynamicPropertySource  
    static void configureProperties(DynamicPropertyRegistry registry) {  
        // Point our service to the fake WireMock server  
        registry.add("notification.service.url", wireMockServer::baseUrl);  
    }

    @Test  
    @DisplayName("Should send a notification after successful enrollment")  
    void testEnrollment\_SendsNotification() {  
        // Arrange  
        Long courseId \= createTestCourse();  
        Long studentId \= 101L;

        // 1\. Stub the external Notification Service  
        wireMockServer.stubFor(post(urlEqualTo("/api/v1/notifications"))  
            .willReturn(aResponse()  
                .withStatus(202)  
                .withHeader("Content-Type", "application/json")  
                .withBody("{\\"status\\":\\"QUEUED\\"}")));

        // 2\. Act: Call our API endpoint that triggers the notification  
        ResponseEntity\<EnrollmentDTO\> response \= restTemplate.postForEntity(  
            "/api/v1/courses/{courseId}/enroll?studentId={studentId}",  
            null,  
            EnrollmentDTO.class,  
            courseId,  
            studentId  
        );

        // 3\. Assert: Check our API's response  
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        // 4\. Verify: Ensure our service called the external API  
        wireMockServer.verify(1, postRequestedFor(urlEqualTo("/api/v1/notifications"))  
            .withRequestBody(containing("\\"studentId\\":101")));  
    }  
}
```

## **7\. CI/CD Integration**

All tests are automated in a CI/CD pipeline to act as a quality gate. This includes not just running tests, but also measuring their *effectiveness*.

* **Test Coverage (Quantity):** We use **JaCoCo** to measure what percentage of our code is *executed* by tests.  
* **Mutation Testing (Quality):** We use **Pitest** to measure test *quality*. Pitest modifies our code (e.g., changing if (a \> b) to if (a \<= b)) and re-runs tests. If no test fails, the "mutant" survives, indicating a gap in our assertions. A high mutation score provides much higher confidence than 100% coverage.

```yml
\# .github/workflows/ci.yml  
name: Spring Boot CI Pipeline

on:  
  pull\_request:  
    branches: \[ main, develop \]

jobs:  
  test:  
    runs-on: ubuntu-latest  
      
    services:  
      postgres:  
        image: postgres:15-alpine  
        env:  
          POSTGRES\_DB: lms\_test  
          POSTGRES\_USER: test  
          POSTGRES\_PASSWORD: test  
        options: \>-  
          \--health-cmd pg\_isready  
          \--health-interval 10s  
          \--health-timeout 5s  
          \--health-retries 5  
      
    steps:  
      \- uses: actions/checkout@v3  
        
      \- name: Set up JDK 21  
        uses: actions/setup-java@v3  
        with:  
          java-version: '21'  
          distribution: 'temurin'  
          cache: 'maven'  
        
      \- name: Run Unit & Slice Tests  
        run: mvn test \-Dtest=\*\*/\*Test  
        
      \- name: Run Full Integration Tests  
        run: mvn verify \-Dtest=\*\*/\*IntegrationTest  
        env:  
          SPRING\_PROFILES\_ACTIVE: test  
        
      \- name: Generate Coverage Report (Quantity Check)  
        run: mvn jacoco:report  
        
      \- name: Enforce Coverage Thresholds  
        run: mvn jacoco:check  
        
      \- name: Upload Coverage to Codecov  
        uses: codecov/codecov-action@v3  
        with:  
          files: ./target/site/jacoco/jacoco.xml  
        
      \- name: Run Mutation Testing (Quality Check)  
        run: mvn org.pitest:pitest-maven:mutationCoverage  
        \# This step will fail the build if mutation score is too low
```

## **8\. Summary**

A production-grade testing strategy for Spring Boot APIs extends far beyond simple unit tests. It requires a multi-layered approach, ideally modeled as a "Testing Honeycomb" that values the integration of components.

This guide outlines a comprehensive strategy that begins with foundational **Unit Tests** using Mockito for isolated logic. It then introduces **Test Slices** (@WebMvcTest, @DataJpaTest) as a critical middle layer to validate component boundaries (like web or persistence) efficiently.

The core of this strategy lies in **Full-Context Integration Tests** (@SpringBootTest) which validate the entire application stack, leveraging **Testcontainers** to ensure true-to-production database testing and **WireMock** to fake external HTTP dependencies.

To operate successfully within a microservices ecosystem, **Contract Testing** (Spring Cloud Contract) is used to prevent breaking changes, while **Performance Testing** (Gatling) and integrated **Security Tests** ensure the application is scalable and resilient.

Finally, this entire strategy is automated within a **CI/CD pipeline** that measures not only test *quantity* (Code Coverage with JaCoCo) but also test *quality* (Mutation Testing with Pitest). Adopting this holistic approach is essential for delivering reliable, secure, and maintainable applications with high confidence.

## Some Good reads:

- https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications
- https://engineering.atspotify.com/2018/01/testing-of-microservices
- https://mvineetsharma.medium.com/software-testing-the-test-pyramid-building-quality-from-the-ground-up-db6528c3802b
