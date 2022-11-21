---
title: Testing Spring Security
date: 2022-11-20
author: Arho Huttunen
summary: Spring Security has good support for MockMvc and WebTestClient. Learn to test authentication and authorization of Spring Boot applications.
categories:
  - Testing
tags:
  - JUnit 5
  - Spring Boot
  - Spring Security
  - Integration testing
image:
  focal_point: center
  preview_only: true
---

Security plays a major role in software. Eventually, everyone needs to add security to their project. In this article, we look at how to test authentication and authorization of Spring Boot applications. We will cover both MVC servlet applications and reactive WebFlux applications.

Spring Security integrates well with the Spring Web MVC and Spring WebFlux frameworks. It also has a comprehensive integration with Spring MVC Test and Spring `WebTestClient`.

{{< toc >}}

Also, if you are interested in a complete course on Spring Boot testing, check out [Testing Spring Boot Applications Masterclass](https://transactions.sendowl.com/stores/13745/226726) by Philip Riecks. You can support me by buying through that link because I get a share.

## Secure a Spring MVC Application

Let's start with a simple application that manages customers. We want to create, get, and delete customers.

```java
@RestController
@RequiredArgsConstructor
public class CustomerController {
    private final CustomerRepository customerRepository;

    @GetMapping("/customer/{id}")
    Customer getCustomer(@PathVariable Long id) {
        return customerRepository.findById(id).orElseThrow();
    }

    @PostMapping("/customer")
    @ResponseStatus(HttpStatus.CREATED)
    Customer createCustomer(@RequestBody Customer customer) {
        return customerRepository.save(customer);
    }

    @DeleteMapping("/customer/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    void deleteCustomer(@PathVariable Long id) {
        customerRepository.deleteById(id);
    }
}
```

We probably don't want unauthorized people to create and delete customers, though. Thus, we are going to use a simple security configuration to add authentication.

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfiguration {
}
```

Now, to secure endpoints, we can use the `@PreAuthorized` annotation to enable method security. We are going to do that for `POST` and `DELETE` operations.

```java
    @PostMapping("/customer")
    @ResponseStatus(HttpStatus.CREATED)
    @PreAuthorize("hasRole('ADMIN')")
    Customer createCustomer(@RequestBody Customer customer) {
        return customerRepository.save(customer);
    }

    @DeleteMapping("/customer/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @PreAuthorize("hasRole('ADMIN')")
    void deleteCustomer(@PathVariable Long id) {
        customerRepository.deleteById(id);
    }
```

From the testing perspective, it doesn't matter much how the security configuration has been setup. For the sake of simplicity, we will keep the example configuration short.

## Set Up @WebMvcTest With Security

To test our controllers in isolation, we can use the Spring Boot Test `@WebMvcTest` test slice.

Using `@WebMvcTest` loads beans needed for the controller, but it doesn't know which other configuration beans to load. We have to tell Spring to load the security configuration by using the `@Import(SecurityConfiguration.class)` annotation.


```java
@WebMvcTest(CustomerController.class)
@Import(SecurityConfiguration.class)
class CustomerControllerTests {
    // ...
}
```

{{% callout warning %}}
If running the application denies access to an endpoint but our tests are not doing the same, it’s likely that we forgot the security configuration import.

This can be hard to spot unless we have tests that test for a forbidden or unauthorized status code.
{{% /callout %}}

{{% callout note %}}
**Additional reading:**

:pencil2: [Testing Web Controllers With @WebMvcTest](/spring-boot-webmvctest/)
{{% /callout %}}

### Run the Test As a User

To test Spring Security, let's start with the endpoints that don't require admin rights.

To run the test as a user, we can use the `@WithMockUser` annotation to provide fake authentication for the user.

```java
    @Test
    @WithMockUser
    void getCustomer() throws Exception {
        when(customerRepository.findById(1L))
                .thenReturn(Optional.of(new Customer(1L, "John", "Doe")));

        mockMvc.perform(get("/customer/{id}", 1L))
                .andExpect(status().isOk());
    }
```

This will make the user authenticated, and the test will pass.

To make sure that our security configuration is working, we can also add a test to verify the response is _401 Unauthorized_ if we haven't authenticated.

We can use the `@WithAnonymousUser` annotation, which is optional but emphasizes the fact it's an unauthenticated user.

```java
    @Test
    @WithAnonymousUser
    void cannotGetCustomerIfNotAuthorized() throws Exception {
        mockMvc.perform(get("/customer/{id}", 1L))
                .andExpect(status().isUnauthorized());
    }
```

In addition to verifying authentication, we can also verify that the response is _403 Forbidden_ if the user is not authorized to access the resource.

```java
    @Test
    @WithMockUser
    void cannotCreateCustomerIfNotAnAdmin() throws Exception {
        mockMvc.perform(post("/customer")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                        .with(csrf())
                )
                .andExpect(status().isForbidden());
    }
```

We can further customize the user and add roles with the `@WithMockUser` annotation. For example, we could make the user an administrator with `@WithMockUser(roles = "ADMIN")`.

### Enable CSRF Token

Spring Security enables CSRF protection by default, which means that we need to add a valid CSRF token to non-safe HTTP methods. These methods include `POST`, `PUT` and `DELETE`, or all the methods that are not read-only.

We can provide a CSRF token with a request post processor.

```java
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanCreateCustomers() throws Exception {
        when(customerRepository.save(any())).thenReturn(new Customer(1L, "John", "Doe"));

        mockMvc.perform(post("/customer")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                        .with(csrf())
                )
                .andExpect(status().isCreated());
    }
```

It's possible to add a non-valid CSRF token with `csrf().useInvalidToken()` but it's not useful unless you are doing a more complex configuration.

### Run As a User Without Annotations

The `@WithMockUser` annotation is handy, but if we don't like annotations, we can use other request post processors instead.

```java
    @Test
    void adminCanDeleteCustomer() throws Exception {
        mockMvc.perform(delete("/customer/{id}", 1L)
                        .with(csrf())
                        .with(user("admin").roles("ADMIN"))
                )
                .andExpect(status().isNoContent());
    }
```

To verify unauthorized status, we can add an `anonymous()` post processor. Using it is optional, but it highlights again the fact that it's an unauthenticated user.

```java
    @Test
    void cannotDeleteCustomerIfNotAuthorized() throws Exception {
        mockMvc.perform(delete("/customer/{id}", 1L)
                        .with(csrf())
                        .with(anonymous())
                )
                .andExpect(status().isUnauthorized());
    }
```

There are multiple other post processors which can be found from `SecurityMockMvcRequestPostProcessors`. For example, we can test HTTP basic authentication, OAuth 2.0 Login, or JWT authentication.

## Set Up MockMvc in @SpringBootTest With Security

If we want to test a larger slice of the application with `@SpringBootTest`, we have to set up the `MockMvc` for the tests.

```java
@SpringBootTest
class CustomerMockEnvTests {
    @Autowired
    private WebApplicationContext context;

    private MockMvc mockMvc;

    @BeforeEach
    public void setup() {
        mockMvc = MockMvcBuilders
                .webAppContextSetup(context)
                .apply(springSecurity())
                .build();
    }
}
```

The important part here is to add `MockMvcConfigurer.springSecurity()` to the configuration.

We could use `@AutoconfigureMockMvc` annotation here instead, but it's good to know how to initialize `MockMvc` manually. The problem with the `@AutoconfigureMockMvc` annotation is that it could mess up with the Spring Boot application context test caching.

Once `MockMvc` has been setup, there is nothing different in using it compared to testing with `@WebMvcTest`. We can also use the `@WithMockUser` annotations.

{{% callout note %}}
**Additional reading:**

:pencil2: [Spring Boot Integration Testing With @SpringBootTest
](/spring-boot-integration-testing/)
{{% /callout %}}

## Set Up MockMvc WebTestClient in @SpringBootTest With Security

What about if we'd like to write end-to-end tests for the application? Since the test will start the application in another process, one option is to use the `WebTestClient` to make requests to the application.

There is one catch though: we cannot auto-wire the `WebTestClient` bean directly in the test. We have to configure the client manually and use a workaround.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class CustomerServerEnvTests {
    @Autowired
    private WebApplicationContext context;

    private WebTestClient webClient;

    @BeforeEach
    void setup() {
        webClient = MockMvcWebTestClient.bindToApplicationContext(context)
                .apply(springSecurity())
                .defaultRequest(get("/").with(csrf()))
                .configureClient()
                .build();
    }

    @Test
    void createCustomer() {
        webClient.mutateWith(csrf()).post().uri("/customer")
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                .exchange()
                .expectStatus().isCreated();
    }
}
```

This is because they originally designed `WebTestClient`  for testing reactive Spring applications and [do not support this yet](https://github.com/spring-projects/spring-security/issues/10841#issuecomment-1048099312).

{{% callout warning %}}
If we try to auto-wire `WebTestClient`, and try to call `mutateWith(csrf())`, tests will fail with a cryptic error.

`Cannot invoke "org.springframework.web.server.adapter.WebHttpHandlerBuilder.filters(java.util.function.Consumer)" because "httpHandlerBuilder" is null`
{{% /callout %}}


Since it's an end to end test, instead of mocking the authentication, we might want to provide the authentication headers instead.

```java
    @Test
    void createCustomer() {
        webClient.post().uri("/customer")
                .headers(http -> http.setBasicAuth("username", "password"))
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                .exchange()
                .expectStatus().isCreated();
    }
```

We can even provide a bearer token with the same mechanism. However, creating tokens is cumbersome, and it's better to focus testing authorization and not representing bearer tokens.

## Secure a Spring WebFlux Application

Let's start with the previous example application and covert that to a reactive application.

```java
@RestController
@RequiredArgsConstructor
public class CustomerController {
    private final CustomerRepository customerRepository;

    @GetMapping("/customer/{id}")
    Mono<Customer> getCustomer(@PathVariable Long id) {
        return customerRepository.findById(id);
    }

    @PostMapping("/customer")
    @ResponseStatus(HttpStatus.CREATED)
    @PreAuthorize("hasRole('ADMIN')")
    Mono<Customer> createCustomer(@RequestBody Customer customer) {
        return customerRepository.save(customer);
    }

    @DeleteMapping("/customer/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @PreAuthorize("hasRole('ADMIN')")
    Mono<Void> deleteCustomer(@PathVariable Long id) {
        return customerRepository.deleteById(id);
    }
}
```

We are going to need to configure security. The annotations differ somewhat from the MVC security configuration.

```java
@Configuration
@EnableWebFluxSecurity
@EnableReactiveMethodSecurity
public class SecurityConfiguration {
}
```

Once the security configuration is in place, the endpoints are now secured.

## Set Up @WebFluxTest With Security

To test our reactive controllers in isolation, we can use the Spring Boot Test `@WebFluxText` test slice. We can also use `WebTestClient` directly in these tests since it has been designed to work with reactive applications.

```java
@WebFluxTest(CustomerController.class)
@Import(SecurityConfiguration.class)
class CustomerControllerTests {
    @MockBean
    private CustomerRepository customerRepository;

    @Autowired
    private WebTestClient webClient;

    @Test
    @WithMockUser
    void getCustomer() {
        when(customerRepository.findById(1L))
                .thenReturn(Mono.just(new Customer(1L, "John", "Doe")));

        webClient.get().uri("/customer/{id}", 1)
                .exchange()
                .expectStatus().isOk();
    }
```

For any endpoints who require the CSRF token, we need to add it. We do this by mutating the `WebTestClient` with mutators coming from `SecurityMockServerConfigurers`.

```java
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanCreateCustomers() {
        when(customerRepository.save(any()))
                .thenReturn(Mono.just(new Customer(1L, "John", "Doe")));

        webClient.mutateWith(csrf())
                .post().uri("/customer")
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"firstName\": \"John\", \"lastName\": \"Doe\"}")
                .exchange()
                .expectStatus().isCreated();
    }
```

Again, if we don't want to use the `@WithMockUser` annotation, we can configure the client further with mutators that provide authentication.

```java
    @Test
    void adminCanDeleteCustomer() {
        when(customerRepository.deleteById(1L)).thenReturn(Mono.empty());

        webClient.mutateWith(csrf())
                .mutateWith(mockUser().roles("ADMIN"))
                .delete().uri("/customer/{id}", 1)
                .exchange()
                .expectStatus().isNoContent();
    }
```

In addition to providing a mock user we can use e.g. `mockJwt()` or `mockOAuth2Login()`.

## Set Up WebFlux WebTestClient in @SpringBooTest With Security

Moving on to end-to-end tests and the `@SpringBootTest`, the `WebTestClient` bean is not available by default. The client needs to be configured manually.

```java
@SpringBootTest
public class CustomerControllerEndToEndTests {
    @Autowired
    private ApplicationContext context;

    private WebTestClient webClient;

    @BeforeEach
    void setup() {
        webClient = WebTestClient.bindToApplicationContext(context)
                .apply(springSecurity())
                .configureClient()
                .build();
    }
}
```

We could again do the same by adding the `@AutoconfigureWebTestClient` annotation, but it suffers from the same problems with context caching as mentioned previously.

Here, it doesn't matter if we are running the test in a mock environment or a server environment. `WebTestClient` is configured the same way in both in a reactive application.

Once `WebTestClient` has been setup there is nothing different in using it compared to testing with `@WebFluxTest`. We can either use the `@WithMockUser` or mutate the client with mock security from `SecurityMockServerConfigurers`.

## Enable Debug Logs For Troubleshooting

Sometimes when our security tests fail, it can be daunting to find out what is wrong. To debug Spring security issues, we can enable security debug logging to see what happens.

```text
logging:
  level:
    org:
      springframework:
        security: DEBUG
```

For example, if we forget the CSRF token, it's not obvious unless we use debug logging. After enabling debug security logging, we can figure out the reason for failure.

```text
o.s.security.web.csrf.CsrfFilter : Invalid CSRF token found for http://localhost/customer
```

## Summary

Spring Security integrates well with the Spring Web MVC and Spring WebFlux frameworks. It also has comprehensive integration with `MockMvc` and `WebTestClient`.

We can fake the authentication using an annotation or a method-based approach. It’s also possible to provide different roles for testing authorization.

It’s not complicated to cover the different cases. The level of verification we want depends on how complex our security configuration is. Since this is a pretty vital part of the application, it is good to test it throughly.

You can find the example code for this article on GitHub [for the MVC application](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-security-testing) and [for the WebFlux application](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-webflux-security-testing).
