# Frameworks summary

## 1. REST

### 1.1 HTTP refresher

- Client-server protocol
- Transport protocol: TCP on port 80
- Request-response

#### HTTP Request

Request types:

- GET
- POST
- PUT
- DELETE
- HEAD
- OPTIONS

Statuscodes:

- 1xx: Informative
- 2xx: Success
- 3xx: Redirection
- 4xx: Client error
- 5xx: Server error

### 1.2 Rest webservices

REST stands for Representational State Transfer. It's an architectural style used for designing networked applications, especially web services. REST relies on a client-server architecture where the client interacts with the server via requests to access and manipulate resources on the server

- **Client-server**
- **Stateless**: server doesn't store any client context
- **Uniform interface**: independent of type of client 
- **Cacheable**: responses from REST service can specify to cache for a specified period. (using the `Cache-Control` header field)
- **Layered system**: client doesn't now which layer it is connected to
- **Code on demand**: Content of message can also be code
- Resource-based: can be any data or service that the app exposes, resources is identifiable via a URI
- CRUD operations: Must support GET, POST, PUT, DELETE methods.
- Representational resources: could be represented by multiple representations (JSON, XML, ...)


#### URI (Uniform Resource Identifier)

URI's are hierarchical structures that represent a resource on which CRUD actions can be performed.

hierarchical: `http://www.myservice.org/discussions/{year}/{day}/{month}/{topic}`

It is also possible to add parameters to the URI to filter or sort on a resources: `http://www.myservice.org/discussions?region=USA&year=2023`

#### Glory of REST

![Glory of REST](./img/glory-of-rest.png)

##### Level 0: The swamp of pox

- One endpoint
- HTTP only as transport
- Remote procedure invocation (`POST <openSlotRequest`)

##### Level 1: Resources

- Multiple endpoints
- Arguments in body (`POST <appointmentRequest`)

##### Level 2: HTTP-methods

- HTTP-methods as CRUD  (`GET ?date=20231224`)
- HTTP-statuscodes as response (`200 OK <openSlotList`)

##### Level 3: Hypermedia controls

- HATEOAS (Hypertext As The Engine Of Application State)
- Links to other resources in response
- Selfdocumenting protocol

### 1.3 Rest webservices: Java (*Springboot*)

**Why springboot?**

Spring avoids a lot of manual configuration and is easy to manage dependently.


#### Spring architecture

![Architecture REST API in Spring](./img/spring-architecture.png)

1. Spring-framework receives HTTP-req and determines which *method* of which *controller* has to get called.
2. (Opt) Convert JSON in body to specific object
3. (Opt) Validate input values of object
4. Call method of *controller* with object as param
5. *Controller* calls underlying logic
6. Logic interacts with data-layer
7. Result of *controller* gets serialized to JSON
8. HTTP-response gets generated (possibly with JSON-object in body)

#### GET controller

```java
import org.springframework.web.bind.annotation.*;

@RestController // Indicate that this class is a RestController
@RequestMapping("agenda") //The path to access agendas (eg: localhost:8080/agenda)
public class AgendaController {
  private Agenda agenda;
  pubic AgendaController(Agenda agenda) { this.agenda = agenda; }

  @GetMapping // Indicate that method handles Get-requests
  public Collection<AgendaItem> agendaFromNow() {
    return agenda.getAgendaItem(new Date());
  }
}
```

- `@RestController`:
  - Normal class
  - Wrapped in a `serverlet`
  - REST-service
- `@RequestMapping`:
  - Specifies which path a REST-service needs to handle.
- `@GetMapping`:
  - Specifies which HTTP-method a method response to
  - Result is automatically converted to JSON
  - Can have a path as parameter (eg `GetMapping("/id/{id}")`)
    - Param can be retrieved using `@RequestParam("id")`
- `@PostMapping`:
  - Body can be retrieved using `@RequestBody AgendaItem item`

##### Dependency injection

DI is needed to make the Agenda class (dependency) available to the to `RestController`. 

```java
@Component
public class Agenda {...}
```

##### Method response status


A REST-method typically returns a `ResponseEntity` (along with a message).

```java
@GetMapping("/age")
ResponseEntity<String> age(@RequestParam("yearOfBirth") int yearOfBirth) {
  if (invalidAge(yearOfBirth)) {
    return new ResponseEntity<>("Invalid year of birth", HttpStatus.BAD_REQUEST);
  }
  return new ResponseEntity<>("YOB is " + yearOfBirth, HttpStatus.OK);
}
```

It is also possible to couple a statuscode to an exception:

```java
@ResponseStatus(code = HttpStatus.BAD_REQUEST)
class CustomException extends RuntimeException {}
```

```java
@PostMapping()
@ResponseStatus(HttpStatus.CREATED)
public void storeEmployee(@RequestBody Employee employee) {...}
```

Here the `ResponseStatus` will automatically return a 201 if the creation of employee was successful.

Finally it is also common to respond with a URI.

```java
@PostMapping("/user/{username}")
public ResponseEntity<Void> createTodo(@PathVariable String username, @RequestBody Todo todo) {
  todo.setUsername(username);
  Todo createdTodo = toJpaRepo.save(todo);
  URI uri = ServletUriComponentsBuilder.fromCurrentRequest()
              .path("/id").buildAndExpand(createdTodo.getId()).toUri();
  return ResponseEntity.created(uri).build();
}
```

### 1.4 Testing webservices

#### Unit tests (Junit)

**Fixture**:

- Fixed context
- baseline state

Four phases:

1. Setup: Prepare text fixture
2. Exercise: Interact with system to research
3. Verify: Verify if it behaves as expected
4. Tear down: Break down text fixture to original state

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
@ActiveProfile("test")
class RestAgendaApplicationTest {
  @Autowired
  private WebTestClient webClient;

  @Test
  public void testGetAllRepos() {
    WebTestClient.get().uri("/api/repos")
      .accept(MediaType.APPLICATION_JSON_UTF8)  // Set header
      .exchange()                               // Send message
      .expectStatus().isOk()                    // Check status
      .expectHeader().contentType(MediaType.APPLICATION_JSON_UTF8) // Check header
      .expectedBodyList(Repo.class)             // Check body
  }
}
```

The `@ActiveProfile` annotation is optional but is very useful. It let's us use a different bean configured specifically for testing purposes.

```java
@Component
@Profile("test")
public class DummyDao implements Dao {...}
```

### 1.5 Webservices security

#### Authentication

Configure login information

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
  @Bean
  public UserDetailsService userDetailsService(BCryptPasswordEncoder bCryptPasswordEncoder) {
    InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
    manager.createUser(User.withUsername("user")
      .password(bCryptPasswordEncoder.encode("userPass"))
      .roles("USER")
      .build());
    manager.createUser(User.withUsername("admin")
      .password(bCryptPasswordEncoder.encode("adminPass"))
      .roles("USER", "ADMIN")
      .build());
    return manager;
  }
...
}
```

#### Authorization

Assign roles to users and restrict certain roles from performing certain actions.

![Authorization](./img/authorization.png)

**Allow certain roles webservice:** (`anyRequest()`)

**Allow certain roles on some paths:** (`requestMatchers(path)`)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
      .authorizeHttpRequests((authz) -> authz
        .requestMatchers("/api/admin/**").hasRole("ADMIN")
        .requestMatchers("/api/user/**").hasRole("USER")
        .anyRequest().authenticated()
      );
    return http.build();
  }
...
}
```

**Allow certain roles on some webservice methods:** (`@EnableMethodSecurity()`)

```java
@Configuration
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true, jsr250Enabled = true)
public class SecurityConfig {
  @RolesAllowed("ROLE_VIEWER")
  public String getUser();

  @RolesAllowed({ "ROLE_ADMIN", "ROLE_EDITOR" })
  public void deleteUser();
}
```

#### CSRF: Cross Site Request Forgery

CSRF is an attack where an attacker tricks a user's browser into making an unintended and unauthorized request to a different website, exploiting the user's authenticated session.

**How to defend against CSRF?**

Make distinction between real and fake HTTP-request by using a **Synchronizer Token Pattern** or **SameSite-attribute cookie**.

Both methods generate a token for the client on login. Then every time the client interacts with the server, the client includes that token in its request.
On every request, the server will validate if it received a valid token. These tokens have a certain expiration date to avoid problems upon token theft.

> 💡: In Spring, CSRF security is enabled by default.

---

## 2. ORM

### 2.1 ORM: Object Relational Mapping

ORM is a programming technique that allows devs to work with objects in the code while transparently persisting and retrieving data from a relational database.
It allows devs to define how objects/classes map to database tables and vice versa. Each object corresponds to a table in the database.

#### Object-relational impedance mismatch

There are a lot of concepts in OOP that resembles the structure of Relational databases:
- Identities
- Relations
- Inheritance

There are a couple requirements to be able to map an object to a DB.

1. Object must have same *Primary Key* as DB table.
2. Relationships should be the same
3. Each field of the object should map to column in the database

#### ORM in java

Using the *JPA (Java Persistence API)*. Every dataclass must be a Java Bean.

```java
@Entity
@Table(name = "sportclub")
public class Sportclub {
  private Long id;
  private String name;

  @Id
  @GeneratedValue(strategy=GenerationType.IDENTITY)
  public Long getId() { return id; }
  public void setId(Long id) { this.id = id;
  }

  public String getName() { return name; }
  public void setName(String name) { this.name = name; }
}
```

There are a couple different strategies to assign a new Primary Key:

* AUTO: Automatically choose strategy
* IDENTITY: Relies on an auto-incremented column in the database
* SEQUENCE: Utilizes a database sequence to generate primary key
* TABLE: Requires an extra table to keep track of the next available id

### 2.2 JPA in Spring

![JPA configuration](./img/jpa-config.png)

#### JPA Repository

The ` JpaRepository` is an interface that is a CRUD-repository with extra functionalities for performing database operations.

```java
// Typically defined in a single file (like ProductRepository.java)
public interface ProductRepository extends JpaRepository<Product, Long> {
      List<Product> findByCategory(String category); // Optional custom user-defined queries
}
```

Example usage of build-in queries:

```java
// ProductDao.java
@Service
public class ProductDao{
  private final ProductRepository repo;

  public ProductDao(ProductRepository repository) {
      this.repo = repository;
  }

  public List<Product> getProductsByCategory(String category) {
    return productRepository.findByCategory(category);
  }

  public Product getProductById(long id) {
    return repo.findById(id).orElse(null);
  }

  public void addProduct(Product p) {
    repo.save(addProduct);
  }
}
```

List of build-in methods:

- `save`: save new object
- `findAll`: return all entities
- `findAllById`: return all with certain id
- `flush`: write changes to DB
- `saveAll`: Save list of objects
- `delete`, `deleteAll`: remove entity/entities
- `existById`: does entity exist?


> ⚠️: Using a JPA requires some configuration in `application.properties`

```text
spring.datasource.url=jdbc:mysql://localhost:3306/dudosdb?serverTimezone=UTC
spring.datasource.username=TLuca
spring.datasource.password=<redacted>
```

### 2.3 ORM Inheritance

There are different annotation to indicate what type of inheritance an object is subject to.

- `@Inheritance(strategy = ...)`: Sets how entities are stored in the DB (default = `SINGLE_TABLE`)
- `@DiscriminatorColumn(name = "...", discriminatorType= ...)`: Defines a column that distinguishes entity types in a single table inheritance strategy.
- `@DiscriminatorValue("K")`: Assigns a specific value to differentiate an entity type in a single table inheritance setup
- `@PrimaryKeyJoinColumn(name = "...")`: Maps a parent table's primary key as a foreign key in the child table in joined table inheritance.


### 2.4 ORM associations: value-objects

Associations:

- Direction:
  - Unidirectional
  - Bidirectional
- Multiplicity:
  - 1-to-1
  - 1-to-many
  - many-to-many

Value-objects criteria:

- Value-obj belongs to just one entity
- Entity disappears -> value-object disappears
- An entity has an *identifier*

#### Value-object example

![Example value-obj](./img/example-value-obj.png)

It doesn't make sense to save the value-object (`Address`) in a separate table. So we can store the `Address` in the same table in multiple `Embedded columns`.

We would also like to make an association to a separate table containing all the email addresses.

```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String state;
    private String zipCode;
    // Getters, setters, constructors, etc.
}

@Entity
@Table(name = "sportclub")
public class sportclub {
  @Id
  private Long id;
  private String name;
  @Embedded
  private Address address;

  @ElementCollection
  @CollectionTable(name = "emailaddresses",
  joinColumns = @JoinColumn(name = "sportclub"))
  @Column(name="email")
}
```

### 2.5 ORM associations: relations

Different mappings of associations of entities:

1. One-to-One Relationship:
  - `@OneToOne`: Defines a one-to-one relationship between two entities.
  - `@JoinColumn`: Specifies the column used for joining two entities in a one-to-one relationship.
2. One-to-Many Relationship:
  - `@OneToMany`: Represents a one-to-many relationship between two entities.
  - `@JoinColumn` (within `@OneToMany`): Specifies the column in the "many" side entity to join with the "one" side entity.
3. Many-to-One Relationship:
  - `@ManyToOne`: Specifies a many-to-one relationship between two entities.
  - `@JoinColumn` (within `@ManyToOne`): Specifies the column in the "many" side entity that references the "one" side entity.
4. Many-to-Many Relationship:
  - `@ManyToMany`: Defines a many-to-many relationship between two entities.
  - `@JoinTable`: Specifies the join table for the many-to-many relationship, defining the columns and their mappings.

For detailed examples of relations (with annotation parameters) see: <https://www.baeldung.com/jpa-hibernate-associations>

### 2.6 How does JPA work?

![JPA operation](./img/jpa-operation.png)

> ❗: TODO

---

