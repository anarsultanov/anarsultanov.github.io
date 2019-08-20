---
layout: post
title: "Database column-level encryption with Spring Data JPA"
author: "Anar Sultanov"
image: /images/posts/2019-08-30/image.jpg
---

Column-level encryption is one of the database encryption methods, which allows user to select specific attributes 
for encryption instead of encrypting the entire database.
<br>In this tutorial, we are going to look at how to implement database column-level encryption with Spring Data JPA.

## 1. Project
We will start by creating a Spring Boot project using Spring Data JPA and H2 embedded database.

#### 1.1. Maven Dependencies
First we need to add the dependencies to `pom.xml`:
```
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```
#### 1.2. Entity
Next, we create an entity class to represent `User` domain object:
{% highlight java %}
@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private String email;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }
}
{% endhighlight %}
#### 1.3. Repository
Now we can create a simple repository which will allow us to perform various operations with `User` objects:
{% highlight java %}
public interface UserRepository extends CrudRepository<User, Long> {}
{% endhighlight %}

Our initial project is ready and we can proceed to the implementation of encryption.

## 2. Attribute Converter
The most optimal solution using JPA would be to implement column-level encryption using the `Attribute Converter` introduced in JPA 2.1.
Another option would be to use JPA listeners, which is available for older versions of JPA and has its advantages, 
but the final implementation is more complex and not reusable, so we will not consider it in this article.

First we need to implement `AttributeConverter`, which will encrypt and decrypt the attributes.
In our implementation, it will use AES algorithm provided by JDK:
{% highlight java %}
@Component
public class AttributeEncryptor implements AttributeConverter<String, String> {

    private static final String AES = "AES";
    private static final String SECRET = "secret-key-12345";

    private final Key key;
    private final Cipher cipher;

    public AttributeEncryptor() throws Exception {
        key = new SecretKeySpec(SECRET.getBytes(), AES);
        cipher = Cipher.getInstance(AES);
    }

    @Override
    public String convertToDatabaseColumn(String attribute) {
        try {
            cipher.init(Cipher.ENCRYPT_MODE, key);
            return Base64.getEncoder().encodeToString(cipher.doFinal(attribute.getBytes()));
        } catch (IllegalBlockSizeException | BadPaddingException | InvalidKeyException e) {
            throw new IllegalStateException(e);
        }
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        try {
            cipher.init(Cipher.DECRYPT_MODE, key);
            return new String(cipher.doFinal(Base64.getDecoder().decode(dbData)));
        } catch (InvalidKeyException | BadPaddingException | IllegalBlockSizeException e) {
            throw new IllegalStateException(e);
        }
    }
}
{% endhighlight %}

Next, we need to add `@Convert` annotation to the attribute that we want to encrypt, and specify the converter:
{% highlight java %}
    @Convert(converter = AttributeEncryptor.class)
    private String name;
{% endhighlight %}

That is all we need to do to implement column-level encryption using `AttributeConverter`.
Let's look at another implementation option and then test them together.

## 3. Database Function
Most databases provide easy to use functions to apply strong encryption algorithms.
Unfortunately, the JPA standard does not offer any options to use these functions, but as long as we adhere to the Spring's default JPA provider, 
which is Hibernate, we can use its proprietary features.
One of these features is `@ColumnTransformer` annotation which allows to provide a custom SQL expression which is used by Hibernate to read the value from and write a value to a column.
We are going to use this annotation to invoke database encryption functions. 
In the case of the H2 database we are using, we will have to use a set of several functions that look pretty ugly:
{% highlight java %}
    @ColumnTransformer(
            read = "TRIM(CHAR(0) FROM UTF8TOSTRING(DECRYPT('AES', HASH('SHA256', STRINGTOUTF8('secret-key-12345'), 1), email)))",
            write = "ENCRYPT('AES', HASH('SHA256', STRINGTOUTF8('secret-key-12345'), 1), STRINGTOUTF8(?))"
    )
    private String email;
{% endhighlight %}

But for most other databases, annotation will look better. For example, when using PostgreSQL:
{% highlight java %}
    @ColumnTransformer(
            read = "PGP_SYM_DECRYPT(email, 'secret-key-12345')",
            write = "PGP_SYM_ENCRYPT (?, 'secret-key-12345')"
    )
    private String email;
{% endhighlight %}
**Note:** A small drawback in this case is that our entity attribute is of type `String`, but PostgreSQL wants to store the encrypted data in a column of type `bytea`,
so this can become an issue for our example since we generate a database table based on the entity, but in any case, for real projects, generating tables in this way is not recommended.

## 4. Test encryption
We set up encryption for the two attributes using different methods, but we need to make sure that they work correctly.
Let’s start by creating a test class that will save a user to the database before running tests and set the user ID in a variable:
{% highlight java %}
@RunWith(SpringRunner.class)
@SpringBootTest
public class AttributeEncryptionTest {

    private static final Logger LOGGER = LoggerFactory.getLogger(AttributeEncryptionTest.class);

    private static final String NAME = "John";
    private static final String EMAIL = "john@example.com";

    private long id;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private UserRepository userRepository;

    @Before
    public void setUp() {
        User user = new User();
        user.setName(NAME);
        user.setEmail(EMAIL);
        id = userRepository.save(user).getId();
    }
}
{% endhighlight %}

Next, we can verify that when we read this user from the database using the repository, all values are in place and are correct:
{% highlight java %}
    @Test
    public void readDecrypted() {
        User user = userRepository.findById(id).orElseThrow();
        assertThat(user.getName()).isEqualTo(NAME);
        assertThat(user.getEmail()).isEqualTo(EMAIL);
    }
{% endhighlight %}

Finally, we can read the values stored in the database, bypassing our repository, and make sure that they do not match our original values:
{% highlight java %}
    @Test
    public void readEncrypted() {
        User user = jdbcTemplate.queryForObject(
                "select * from user where id = ?",
                (resultSet, i) -> {
                    User result = new User();
                    result.setId(resultSet.getLong("id"));
                    result.setName(resultSet.getString("name"));
                    result.setEmail(resultSet.getString("email"));
                    return result;
                },
                id
        );

        assertThat(user.getName()).isNotEqualTo(NAME);
        LOGGER.info("Encrypted name value in DB is {}", user.getName());
        assertThat(user.getEmail()).isNotEqualTo(EMAIL);
        LOGGER.info("Encrypted email value in DB is {}", user.getEmail());
    }
{% endhighlight %}

And to satisfy our curiosity, we also output the values from the database to the console:
{% highlight console %}
...
INFO 45832 --- [           main] d.s.s.a.AttributeEncryptionTest          : Encrypted name value in DB is j3gn4/JlF4TfLlmmkwPIpg==
INFO 45832 --- [           main] d.s.s.a.AttributeEncryptionTest          : Encrypted email value in DB is 2e82bf3aeaa3b8f0805d9cf9d4fb8b87
...
{% endhighlight %}

## 5. Conclusion
We learned how to implement column-level encryption in several ways, 
but we must not forget about the risks associated with this process, most of which are related to key management.
For example, insecure storage of keys can give people with malicious intent the ability to decrypt sensitive data using keys that they have access to, 
and the loss of keys in most cases entails data loss, since decryption without keys is almost impossible.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/spring-data-attribute-encryption)._
