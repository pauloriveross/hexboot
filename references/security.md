# Security

## @PreAuthorize placement
- On **inbound port** interface, NOT on controller, NOT on use case implementation
- Reason: security is a cross-cutting concern of the application layer, not of HTTP

```java
public interface CreateOrderUseCase {
  @PreAuthorize("hasRole('ADMIN')")
  Order create(CreateOrderCommand command);
}
```

## Security configuration
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
      .csrf(csrf -> csrf.disable())
      .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/v1/**").authenticated()
        .anyRequest().permitAll())
      .addFilterBefore(jwtFilter(), UsernamePasswordAuthenticationFilter.class)
      .build();
  }

  @Bean
  public JwtAuthenticationFilter jwtFilter() {
    return new JwtAuthenticationFilter();
  }
}
```

## JWT filter (adapter/inbound/security/JwtAuthenticationFilter.java)
```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
      throws ServletException, IOException {
    var token = extractToken(request);
    if (token != null) {
      // validate token, set SecurityContext
    }
    chain.doFilter(request, response);
  }

  private String extractToken(HttpServletRequest request) {
    var header = request.getHeader("Authorization");
    if (header != null && header.startsWith("Bearer ")) return header.substring(7);
    return null;
  }
}
```

## OAuth2 resource server
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${JWT_ISSUER_URI}
```
