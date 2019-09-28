---
layout: post
title: "Schema-based multi-tenancy with Spring Data, Hibernate and Flyway"
author: "Anar Sultanov"
image: /images/posts/2019-09-28/image.jpg
---

Multi-tenancy is an architectural pattern that allows multiple tenants to use a single instance of software,
the purpose of which is to provide each tenant with a dedicated share of the instance but to isolate the information belonging to each tenant.
<br>In this tutorial, we are going to look at how to implement schema-based multi-tenancy in Spring Boot application.

## 1. Project
We will start by creating a simple RESTful web service, protected by Spring Security, that will use Spring Data JPA to persist data in the embedded H2 database.

#### 1.1. Maven Dependencies
First, we add the necessary dependencies to `pom.xml`:
```
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
```
#### 1.2. Entities
Next, we create entity classes that represent our domain objects:
{% highlight java %}
@Entity
public class User implements UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String username;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private String password;

    // getters, setters and overriden methods from UserDetails
}
{% endhighlight %}
{% highlight java %}
@Entity
public class Note {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String text;
    
    // getters and setters
}
{% endhighlight %}

#### 1.3. Repositories
Then we define CRUD repositories for the entities:
{% highlight java %}
@Repository
public interface UserRepository extends CrudRepository<User, Long> {
    Optional<User> findByUsername(String username);
}
{% endhighlight %}
{% highlight java %}
@Repository
public interface NoteRepository extends CrudRepository<Note, Long> {
}
{% endhighlight %}

#### 1.4. Services
Now we can implement some logic in the service layer:
{% highlight java %}
@Service
public class UserService implements UserDetailsService {

    private UserRepository repository;
    private PasswordEncoder encoder;

    public UserService(UserRepository repository) {
        this.repository = repository;
        this.encoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }

    public User createUser(User user) {
        String encodedPassword = encoder.encode(user.getPassword());
        user.setPassword(encodedPassword);
        return repository.save(user);
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return repository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User with the specified username is not found"));
    }
}
{% endhighlight %}
{% highlight java %}
@Service
public class NoteService {

    private NoteRepository repository;

    public NoteService(NoteRepository repository) {
        this.repository = repository;
    }

    public Note createNote(Note note) {
        return repository.save(note);
    }

    public Note findNote(Long id) {
        return repository.findById(id).orElseThrow();
    }

    public Iterable<Note> findAllNotes() {
        return repository.findAll();
    }
}
{% endhighlight %}

#### 1.5. Controllers
Finally, we can expose our web service with REST API:
{% highlight java %}
@RestController
@RequestMapping("/users")
public class UserController {

    private UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public ResponseEntity<User> register(@RequestBody User user) {
        User created = userService.createUser(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
{% endhighlight %}
{% highlight java %}
@RestController
@RequestMapping("/notes")
public class NoteController {

    private NoteService noteService;

    public NoteController(NoteService noteService) {
        this.noteService = noteService;
    }

    @PostMapping
    public ResponseEntity<Note> createNote(@RequestBody Note note) {
        Note created = noteService.createNote(note);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Note> getNote(@PathVariable Long id) {
        Note note = noteService.findNote(id);
        return ResponseEntity.ok(note);
    }

    @GetMapping
    public ResponseEntity<Iterable<Note>> getAllNotes() {
        Iterable<Note> notes = noteService.findAllNotes();
        return ResponseEntity.ok(notes);
    }
}
{% endhighlight %}

#### 1.6. Security
The web service is ready and we can run it, but all of our API endpoints are publicly available. 
To protect some of them, we need to configure http security in a configuration class that extends `WebSecurityConfigurerAdapter`.
We will make the user registration endpoint public, and also open access to the H2 console, which can be enabled using the `spring.h2.console.enabled = true` parameter.
All other endpoints will require authentication, but in order not to complicate the example with some token-based authentication, we just enable http basic authentication for them:
{% highlight java %}
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private UserDetailsService userService;

    public SecurityConfig(UserDetailsService userService) {
        this.userService = userService;
    }

    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userService);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/h2-console/**").permitAll()
                .antMatchers(HttpMethod.POST, "/users").permitAll()
                .anyRequest().authenticated()
                .and()
                .httpBasic()
                .and()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .csrf().disable()
                .headers().frameOptions().disable();
    }
}
{% endhighlight %}
Note that we use the `UserService` to provide the authentication manager with user details.
For this purpose, we made it to implement `UserDetailsService`, and we made our `User` class to implement `UserDetails`.

## 2. Multi-tenancy
Typically, a tenant would be a group of users, such as an organization, but in our simplified implementation, each user will be a separate tenant.

Also you probably already noticed that, although we can create notes on behalf of different users, all of them are saved in one table, and all users have access to all notes.
The obvious solution would be to record the owner of the notes in the database and retrieve the notes for each user using this data.
This is the most common multi-tenancy option, but since we want more isolation between tenant data, this option does not suit us. 
We would get the greatest isolation when each tenant uses a separate database, but this complicates and increases the cost of our infrastructure.
But there is a third option that is cheaper and still provides partial isolation: the schema-based multi-tenancy.

#### 2.1. Hibernate configuration
We are going to use the multi-tenancy features of Hibernate, which is the default JPA provider in Spring Data.
All we need to do is provide an implementation of the `CurrentTenantIdentifierResolver` and `MultiTenantConnectionProvider` interfaces,
and add them to the JPA properties along with the multi-tenancy strategy.

`CurrentTenantIdentifierResolver` - resolves the tenant identifier to use. 
In our implementation, it gets the authentication data from the security context and uses the username as the identifier of the tenant; 
if authentication is anonymous or missing, it falls back to the default tenant identifier.
{% highlight java %}
@Component
public class TenantIdentifierResolver implements CurrentTenantIdentifierResolver {

    static final String DEFAULT_TENANT = "default";

    @Override
    public String resolveCurrentTenantIdentifier() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .filter(Predicate.not(authentication -> authentication instanceof AnonymousAuthenticationToken))
                .map(Principal::getName)
                .orElse(DEFAULT_TENANT);
    }

    @Override
    public boolean validateExistingCurrentSessions() {
        return true;
    }
}
{% endhighlight %}

`MultiTenantConnectionProvider` – provides connections based on tenant identifier.
Our implementation reuses the JDBC connection pool to serve all tenants, but before using the `Connection`, 
it alters it with the `SET SCHEMA` command to reference the schema named by the tenant identifier.
{% highlight java %}
@Component
public class TenantConnectionProvider implements MultiTenantConnectionProvider {

    private DataSource datasource;

    public TenantConnectionProvider(DataSource dataSource) {
        this.datasource = dataSource;
    }

    @Override
    public Connection getAnyConnection() throws SQLException {
        return datasource.getConnection();
    }

    @Override
    public void releaseAnyConnection(Connection connection) throws SQLException {
        connection.close();
    }

    @Override
    public Connection getConnection(String tenantIdentifier) throws SQLException {
        final Connection connection = getAnyConnection();
        connection.createStatement()
                .execute(String.format("SET SCHEMA \"%s\";", tenantIdentifier));
        return connection;
    }

    @Override
    public void releaseConnection(String tenantIdentifier, Connection connection) throws SQLException {
        connection.createStatement()
                .execute(String.format("SET SCHEMA \"%s\";", TenantIdentifierResolver.DEFAULT_TENANT));
        releaseAnyConnection(connection);
    }

    @Override
    public boolean supportsAggressiveRelease() {
        return false;
    }

    @Override
    public boolean isUnwrappableAs(Class unwrapType) {
        return false;
    }

    @Override
    public <T> T unwrap(Class<T> unwrapType) {
        return null;
    }
}
{% endhighlight %}

It remains only to set all the necessary parameters in the configuration:
{% highlight java %}
@Configuration
public class HibernateConfig {

    @Bean
    public JpaVendorAdapter jpaVendorAdapter() {
        return new HibernateJpaVendorAdapter();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(DataSource dataSource, JpaProperties jpaProperties,
            MultiTenantConnectionProvider multiTenantConnectionProvider, CurrentTenantIdentifierResolver tenantIdentifierResolver) {

        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan(MultiTenancyApplication.class.getPackage().getName());
        em.setJpaVendorAdapter(jpaVendorAdapter());

        Map<String, Object> jpaPropertiesMap = new HashMap<>(jpaProperties.getProperties());
        jpaPropertiesMap.put(Environment.MULTI_TENANT, MultiTenancyStrategy.SCHEMA);
        jpaPropertiesMap.put(Environment.MULTI_TENANT_CONNECTION_PROVIDER, multiTenantConnectionProvider);
        jpaPropertiesMap.put(Environment.MULTI_TENANT_IDENTIFIER_RESOLVER, tenantIdentifierResolver);
        em.setJpaPropertyMap(jpaPropertiesMap);

        return em;
    }
}
{% endhighlight %}

#### 2.2. Flyway configuration
The tables in the shared schema for unauthenticated users and in the tenant schemas will be different, 
so we need a way to perform different migrations in these schemas, and we will use Flyway for this.
We configure it to start migrations for the default schema from the `db/migration/default` directory, 
and then iterate over all tenants and start migrations for each of them from the `db/migration/tenants` directory.
{% highlight java %}
@Configuration
public class FlywayConfig {

    @Bean
    public Flyway flyway(DataSource dataSource) {
        Flyway flyway = Flyway.configure()
                .locations("db/migration/default")
                .dataSource(dataSource)
                .schemas(TenantIdentifierResolver.DEFAULT_TENANT)
                .load();
        flyway.migrate();
        return flyway;
    }

    @Bean
    CommandLineRunner commandLineRunner(UserRepository repository, DataSource dataSource) {
        return args -> {
            repository.findAll().forEach(user -> {
                String tenant = user.getUsername();
                Flyway flyway = Flyway.configure()
                        .locations("db/migration/tenants")
                        .dataSource(dataSource)
                        .schemas(tenant)
                        .load();
                flyway.migrate();
            });
        };
    }
}
{% endhighlight %}

Next, we will create the necessary migrations in the appropriate directories:
{% highlight sql %}
-- db/migration/default/V1__init_schema.sql
CREATE TABLE user
(
    id                 BIGINT AUTO_INCREMENT,
    username           VARCHAR(255) UNIQUE,
    password           VARCHAR(255)
);
{% endhighlight %}
{% highlight sql %}
-- db/migration/tenants/V1__init_schema.sql
CREATE TABLE note
(
    id                 BIGINT AUTO_INCREMENT,
    text               TEXT
);
{% endhighlight %}

We also create a service for programmatically creating a scheme for new tenants and performing all relevant migrations:
{% highlight java %}
@Component
public class TenantService {

    private DataSource dataSource;

    public TenantService(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void initDatabase(String schema) {
        Flyway flyway = Flyway.configure()
                .locations("db/migration/tenants")
                .dataSource(dataSource)
                .schemas(schema)
                .load();
        flyway.migrate();
    }
}
{% endhighlight %}

Finally, we update our user service and make it initiate the creation of a schema for new users:
{% highlight java %}
@Service
public class UserService implements UserDetailsService {

    // ...
    
    @Transactional
    public User createUser(User user) {
        String encodedPassword = encoder.encode(user.getPassword());
        user.setPassword(encodedPassword);
        User saved = repository.save(user);
        tenantService.initDatabase(user.getUsername());
        return saved;
    }
}
{% endhighlight %}

## 3. Test
Now we can run our application and test it. We start by creating multiple users:
{% highlight console %} 
> curl -X POST -H "Content-Type: application/json" -d "{\"username\":\"john\",\"password\":\"password\"}" http://localhost:8080/users
{"id":1,"username":"john"}
> curl -X POST -H "Content-Type: application/json" -d "{\"username\":\"jane\",\"password\":\"qwerty123\"}" http://localhost:8080/users
{"id":2,"username":"jane"}
{% endhighlight %}

We create a note using John's credentials:
{% highlight console %} 
> curl -u john:password -X POST -H "Content-Type: application/json" -d "{\"text\":\"Hello from John!\"}" http://localhost:8080/notes
{"id":1,"text":"Hello from John!"}
{% endhighlight %}

Then we also create a note using Jane's credentials:
{% highlight console %} 
> curl -u jane:qwerty123 -X POST -H "Content-Type: application/json" -d "{\"text\":\"Hello from Jane!\"}" http://localhost:8080/notes
{"id":1,"text":"Hello from Jane!"}
{% endhighlight %}
We can notice that the generated note ID is the same because notes were stored in separate tables in schemas specific to the tenant.

Finally, we will request all notes using the credentials of one of the users and make sure that we only get notes belonging to this user:
{% highlight console %} 
> curl -u jane:qwerty123 http://localhost:8080/notes
[{"id":1,"text":"Hello from Jane!"}]
{% endhighlight %}

Also, to make sure that everything works as we intended, 
we can go to [http://localhost:8080/h2-console](http://localhost:8080/h2-console) and use the H2 console, 
access to which we opened earlier.

## 4. Conclusion
Schema-based multi-tenancy provides the best balance between performance, tenant isolation, development complexity, and infrastructure cost.
However, when choosing a multi-tenancy strategy, you should always consider the level of security that your tenants will require 
and indicate the level of data isolation that you provide in the Service-Level Agreement: either full or partial.
Besides, sharing resources in a schema-based approach can make SLA compliance difficult, 
as one tenant performing resource-intensive tasks can cause latency spikes for everyone else.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/schema-based-multi-tenancy)._
