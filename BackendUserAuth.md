

This guide provides two methods for implementing user authentication in a Spring Boot application:
1. **Real-Time Authentication with a Database** (production-ready).
2. **In-Memory Authentication Using HashMap** (for learning/testing purposes).

---

## 1. Real-Time Authentication with Database (Production-Ready)

This approach uses **Spring Security**, **JWT (JSON Web Token)**, and **JPA with a database** for persistent user storage.

## Step 1: Set Up Database with JPA

### 1) Define a `User` entity for storing user information in the database.

```java
import javax.persistence.*;

@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String email;
    private String password; // Store hashed passwords

    // Constructors, Getters, Setters
}
```
### 2) Repository
```java
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}
```
### 3) Implement AuthService with Database Authentication
```java
import java.util.Optional;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Service;

@Service
public class AuthService {
    @Autowired
    private UserRepository userRepository;
    private BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
    private JwtUtil jwtUtil = new JwtUtil();

    public String registerUser(String username, String email, String password) {
        if (userRepository.findByUsername(username).isPresent()) {
            throw new RuntimeException("User already exists");
        }
        User user = new User();
        user.setUsername(username);
        user.setEmail(email);
        user.setPassword(passwordEncoder.encode(password));
        userRepository.save(user);
        return "User registered successfully";
    }

    public String loginUser(String username, String password) {
        Optional<User> user = userRepository.findByUsername(username);
        if (user.isPresent() && passwordEncoder.matches(password, user.get().getPassword())) {
            return jwtUtil.generateToken(username);
        } else {
            throw new RuntimeException("Invalid credentials");
        }
    }
}
```
### 4)JWT Utility Class for Token Generation
```java
import io.jsonwebtoken.*;
import java.util.Date;
import org.springframework.stereotype.Component;

@Component
public class JwtUtil {
    private final String SECRET_KEY = "yourSecretKey";

    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 1000 * 60 * 60 * 10)) // 10 hours
            .signWith(SignatureAlgorithm.HS256, SECRET_KEY)
            .compact();
    }

    public String extractUsername(String token) {
        return Jwts.parser().setSigningKey(SECRET_KEY).parseClaimsJws(token).getBody().getSubject();
    }

    public Boolean isTokenValid(String token, String username) {
        final String extractedUsername = extractUsername(token);
        return (extractedUsername.equals(username) && !isTokenExpired(token));
    }

    private Boolean isTokenExpired(String token) {
        return Jwts.parser().setSigningKey(SECRET_KEY).parseClaimsJws(token).getBody().getExpiration().before(new Date());
    }
}
```
## 2. In-Memory Authentication Using HashMap (For Testing)
### This approach uses an in-memory HashMap to simulate database storage, suitable for testing purposes but lacks persistence and security.

```java
import java.util.HashMap;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Service;

@Service
public class AuthServiceHashMap {
    private HashMap<String, String> users = new HashMap<>(); // Simulate database
    private BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
    private JwtUtil jwtUtil = new JwtUtil();

    public String registerUser(String username, String password) {
        if (users.containsKey(username)) {
            throw new RuntimeException("User already exists");
        }
        users.put(username, passwordEncoder.encode(password));
        return "User registered successfully";
    }

    public String loginUser(String username, String password) {
        String hashedPassword = users.get(username);
        if (hashedPassword != null && passwordEncoder.matches(password, hashedPassword)) {
            return jwtUtil.generateToken(username); // Return JWT token
        } else {
            throw new RuntimeException("Invalid credentials");
        }
    }
}
```


