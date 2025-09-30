# Phase 7: 보안 및 운영 기능 - 구현 코드

## 개요
사용자 인증, 권한 관리, 시스템 설정 관리, 로깅 및 모니터링 기능을 구현합니다.

## 1. Spring Security 설정

### 1.1 SecurityConfig.java
**파일 위치**: `/src/main/java/com/sbs/datasync/config/SecurityConfig.java`
```java
package com.sbs.datasync.config;

import com.sbs.datasync.security.JwtAuthenticationEntryPoint;
import com.sbs.datasync.security.JwtAuthenticationFilter;
import com.sbs.datasync.security.CustomUserDetailsService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.Arrays;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    private final CustomUserDetailsService userDetailsService;
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    public SecurityConfig(CustomUserDetailsService userDetailsService,
                         JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint,
                         JwtAuthenticationFilter jwtAuthenticationFilter) {
        this.userDetailsService = userDetailsService;
        this.jwtAuthenticationEntryPoint = jwtAuthenticationEntryPoint;
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .exceptionHandling(ex -> ex.authenticationEntryPoint(jwtAuthenticationEntryPoint))
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(authz -> authz
                // Public endpoints
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/health/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/swagger-ui/**", "/api-docs/**").permitAll()

                // Admin endpoints
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/system/**").hasRole("ADMIN")

                // Batch management - requires OPERATOR or ADMIN role
                .requestMatchers("/api/batch/jobs/**").hasAnyRole("OPERATOR", "ADMIN")
                .requestMatchers("/api/batch/execution/**").hasAnyRole("OPERATOR", "ADMIN")

                // Schedule management - requires SCHEDULER or ADMIN role
                .requestMatchers("/api/batch/schedule/**").hasAnyRole("SCHEDULER", "ADMIN")

                // History and monitoring - requires VIEWER role or above
                .requestMatchers("/api/batch/history/**").hasAnyRole("VIEWER", "OPERATOR", "SCHEDULER", "ADMIN")
                .requestMatchers("/api/batch/status/**").hasAnyRole("VIEWER", "OPERATOR", "SCHEDULER", "ADMIN")

                // All other requests require authentication
                .anyRequest().authenticated()
            );

        http.authenticationProvider(authenticationProvider());
        http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(Arrays.asList("http://localhost:*", "https://*.company.com"));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(Arrays.asList("authorization", "content-type", "x-auth-token"));
        configuration.setExposedHeaders(Arrays.asList("x-auth-token"));
        configuration.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

### 1.2 User 엔티티
**파일 위치**: `/src/main/java/com/sbs/datasync/entity/User.java`
```java
package com.sbs.datasync.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "USERS",
       uniqueConstraints = {
           @UniqueConstraint(columnNames = "username"),
           @UniqueConstraint(columnNames = "email")
       })
public class User extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "USER_ID")
    private Long userId;

    @NotBlank
    @Size(max = 50)
    @Column(name = "USERNAME", nullable = false, length = 50)
    private String username;

    @NotBlank
    @Size(max = 100)
    @Email
    @Column(name = "EMAIL", nullable = false, length = 100)
    private String email;

    @NotBlank
    @Size(max = 100)
    @Column(name = "PASSWORD", nullable = false, length = 100)
    private String password;

    @Size(max = 100)
    @Column(name = "FULL_NAME", length = 100)
    private String fullName;

    @Column(name = "IS_ENABLED", nullable = false)
    private Boolean isEnabled = true;

    @Column(name = "IS_ACCOUNT_NON_EXPIRED", nullable = false)
    private Boolean isAccountNonExpired = true;

    @Column(name = "IS_ACCOUNT_NON_LOCKED", nullable = false)
    private Boolean isAccountNonLocked = true;

    @Column(name = "IS_CREDENTIALS_NON_EXPIRED", nullable = false)
    private Boolean isCredentialsNonExpired = true;

    @Column(name = "LAST_LOGIN_DATE")
    private LocalDateTime lastLoginDate;

    @Column(name = "FAILED_LOGIN_ATTEMPTS", nullable = false)
    private Integer failedLoginAttempts = 0;

    @Column(name = "ACCOUNT_LOCKED_DATE")
    private LocalDateTime accountLockedDate;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "USER_ROLES",
               joinColumns = @JoinColumn(name = "USER_ID"),
               inverseJoinColumns = @JoinColumn(name = "ROLE_ID"))
    private Set<Role> roles = new HashSet<>();

    // Constructors, getters, setters...

    public boolean hasRole(String roleName) {
        return roles.stream().anyMatch(role -> role.getName().equals(roleName));
    }

    public boolean hasAnyRole(String... roleNames) {
        return roles.stream().anyMatch(role ->
            Arrays.stream(roleNames).anyMatch(roleName -> role.getName().equals(roleName)));
    }

    public void incrementFailedLoginAttempts() {
        this.failedLoginAttempts++;
        if (this.failedLoginAttempts >= 5) {
            this.isAccountNonLocked = false;
            this.accountLockedDate = LocalDateTime.now();
        }
    }

    public void resetFailedLoginAttempts() {
        this.failedLoginAttempts = 0;
        this.isAccountNonLocked = true;
        this.accountLockedDate = null;
    }

    public void updateLastLoginDate() {
        this.lastLoginDate = LocalDateTime.now();
    }
}
```

### 1.3 JWT 토큰 처리
**파일 위치**: `/src/main/java/com/sbs/datasync/security/JwtUtils.java`
```java
package com.sbs.datasync.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.util.Date;

@Component
public class JwtUtils {

    @Value("${app.jwt.secret:defaultSecretKeyThatShouldBeChangedInProduction}")
    private String jwtSecret;

    @Value("${app.jwt.expiration:86400}")
    private int jwtExpirationMs;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(jwtSecret.getBytes());
    }

    public String generateJwtToken(Authentication authentication) {
        UserDetails userPrincipal = (UserDetails) authentication.getPrincipal();

        return Jwts.builder()
                .setSubject(userPrincipal.getUsername())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + jwtExpirationMs * 1000L))
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }

    public String getUsernameFromToken(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody()
                .getSubject();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (MalformedJwtException e) {
            logger.error("Invalid JWT token: {}", e.getMessage());
        } catch (ExpiredJwtException e) {
            logger.error("JWT token is expired: {}", e.getMessage());
        } catch (UnsupportedJwtException e) {
            logger.error("JWT token is unsupported: {}", e.getMessage());
        } catch (IllegalArgumentException e) {
            logger.error("JWT claims string is empty: {}", e.getMessage());
        }
        return false;
    }

    public Date getExpirationDateFromToken(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody()
                .getExpiration();
    }
}
```

## 2. 시스템 설정 관리

### 2.1 SystemConfig 엔티티
**파일 위치**: `/src/main/java/com/sbs/datasync/entity/SystemConfig.java`
```java
package com.sbs.datasync.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

@Entity
@Table(name = "SYSTEM_CONFIG",
       indexes = {
           @Index(name = "IDX_SYSTEM_CONFIG_01", columnList = "CONFIG_CATEGORY"),
           @Index(name = "IDX_SYSTEM_CONFIG_02", columnList = "CONFIG_KEY")
       })
public class SystemConfig extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "CONFIG_ID")
    private Long configId;

    @NotBlank
    @Size(max = 50)
    @Column(name = "CONFIG_CATEGORY", nullable = false, length = 50)
    private String configCategory;

    @NotBlank
    @Size(max = 100)
    @Column(name = "CONFIG_KEY", nullable = false, length = 100)
    private String configKey;

    @NotBlank
    @Size(max = 1000)
    @Column(name = "CONFIG_VALUE", nullable = false, length = 1000)
    private String configValue;

    @Size(max = 500)
    @Column(name = "DESCRIPTION", length = 500)
    private String description;

    @Column(name = "IS_ENCRYPTED", nullable = false)
    private Boolean isEncrypted = false;

    @Column(name = "IS_EDITABLE", nullable = false)
    private Boolean isEditable = true;

    @Size(max = 50)
    @Column(name = "DATA_TYPE", length = 50)
    private String dataType = "STRING"; // STRING, INTEGER, BOOLEAN, DECIMAL

    @Size(max = 500)
    @Column(name = "VALIDATION_RULE", length = 500)
    private String validationRule;

    // Constructors, getters, setters...

    public enum ConfigCategory {
        DATABASE("데이터베이스"),
        BATCH("배치 처리"),
        NOTIFICATION("알림"),
        SECURITY("보안"),
        PERFORMANCE("성능"),
        LOGGING("로깅");

        private final String description;

        ConfigCategory(String description) {
            this.description = description;
        }

        public String getDescription() {
            return description;
        }
    }

    public Object getTypedValue() {
        switch (dataType.toUpperCase()) {
            case "INTEGER":
                return Integer.valueOf(configValue);
            case "BOOLEAN":
                return Boolean.valueOf(configValue);
            case "DECIMAL":
                return Double.valueOf(configValue);
            default:
                return configValue;
        }
    }
}
```

### 2.2 SystemConfigService.java
**파일 위치**: `/src/main/java/com/sbs/datasync/service/SystemConfigService.java`
```java
package com.sbs.datasync.service;

import com.sbs.datasync.entity.SystemConfig;
import com.sbs.datasync.repository.SystemConfigRepository;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
@Transactional(readOnly = true)
public class SystemConfigService {

    private final SystemConfigRepository configRepository;
    private final SecretKey encryptionKey;

    public SystemConfigService(SystemConfigRepository configRepository) {
        this.configRepository = configRepository;
        this.encryptionKey = generateEncryptionKey();
    }

    @Cacheable(value = "systemConfig", key = "#category + '_' + #key")
    public String getConfigValue(String category, String key) {
        return configRepository.findByCategoryAndKey(category, key)
                .map(config -> {
                    if (config.getIsEncrypted()) {
                        return decrypt(config.getConfigValue());
                    }
                    return config.getConfigValue();
                })
                .orElse(null);
    }

    @Cacheable(value = "systemConfig", key = "#category + '_' + #key + '_typed'")
    public <T> T getConfigValue(String category, String key, Class<T> type) {
        SystemConfig config = configRepository.findByCategoryAndKey(category, key)
                .orElse(null);

        if (config == null) {
            return null;
        }

        String value = config.getIsEncrypted() ? decrypt(config.getConfigValue()) : config.getConfigValue();

        return convertValue(value, type);
    }

    public Map<String, String> getConfigsByCategory(String category) {
        return configRepository.findByCategory(category)
                .stream()
                .collect(Collectors.toMap(
                    SystemConfig::getConfigKey,
                    config -> config.getIsEncrypted() ? decrypt(config.getConfigValue()) : config.getConfigValue()
                ));
    }

    @Transactional
    @CacheEvict(value = "systemConfig", key = "#category + '_' + #key")
    public void updateConfig(String category, String key, String value) {
        SystemConfig config = configRepository.findByCategoryAndKey(category, key)
                .orElseThrow(() -> new IllegalArgumentException("설정을 찾을 수 없습니다: " + category + "." + key));

        if (!config.getIsEditable()) {
            throw new IllegalStateException("편집 불가능한 설정입니다: " + category + "." + key);
        }

        // 검증 규칙 확인
        validateConfigValue(config, value);

        String valueToStore = config.getIsEncrypted() ? encrypt(value) : value;
        config.setConfigValue(valueToStore);

        configRepository.save(config);

        // 설정 변경 이벤트 발행
        applicationEventPublisher.publishEvent(new ConfigChangedEvent(category, key, value));
    }

    @Transactional
    public void createConfig(SystemConfig config) {
        if (configRepository.existsByCategoryAndKey(config.getConfigCategory(), config.getConfigKey())) {
            throw new IllegalArgumentException("이미 존재하는 설정입니다: " +
                config.getConfigCategory() + "." + config.getConfigKey());
        }

        if (config.getIsEncrypted() && config.getConfigValue() != null) {
            config.setConfigValue(encrypt(config.getConfigValue()));
        }

        configRepository.save(config);
    }

    // 기본 설정값들 초기화
    @PostConstruct
    public void initializeDefaultConfigs() {
        createDefaultConfigIfNotExists("BATCH", "DEFAULT_CHUNK_SIZE", "100",
            "기본 청크 크기", "INTEGER", false);
        createDefaultConfigIfNotExists("BATCH", "MAX_THREAD_POOL_SIZE", "8",
            "최대 스레드 풀 크기", "INTEGER", false);
        createDefaultConfigIfNotExists("DATABASE", "CONNECTION_TIMEOUT", "30000",
            "DB 연결 타임아웃(ms)", "INTEGER", false);
        createDefaultConfigIfNotExists("NOTIFICATION", "EMAIL_ENABLED", "true",
            "이메일 알림 사용", "BOOLEAN", false);
        createDefaultConfigIfNotExists("NOTIFICATION", "SMTP_PASSWORD", "",
            "SMTP 패스워드", "STRING", true);
    }

    private void createDefaultConfigIfNotExists(String category, String key, String value,
                                              String description, String dataType, boolean encrypted) {
        if (!configRepository.existsByCategoryAndKey(category, key)) {
            SystemConfig config = new SystemConfig();
            config.setConfigCategory(category);
            config.setConfigKey(key);
            config.setConfigValue(encrypted ? encrypt(value) : value);
            config.setDescription(description);
            config.setDataType(dataType);
            config.setIsEncrypted(encrypted);
            configRepository.save(config);
        }
    }

    private String encrypt(String plainText) {
        try {
            Cipher cipher = Cipher.getInstance("AES");
            cipher.init(Cipher.ENCRYPT_MODE, encryptionKey);
            byte[] encrypted = cipher.doFinal(plainText.getBytes());
            return Base64.getEncoder().encodeToString(encrypted);
        } catch (Exception e) {
            throw new RuntimeException("암호화 실패", e);
        }
    }

    private String decrypt(String encryptedText) {
        try {
            Cipher cipher = Cipher.getInstance("AES");
            cipher.init(Cipher.DECRYPT_MODE, encryptionKey);
            byte[] decrypted = cipher.doFinal(Base64.getDecoder().decode(encryptedText));
            return new String(decrypted);
        } catch (Exception e) {
            throw new RuntimeException("복호화 실패", e);
        }
    }

    private SecretKey generateEncryptionKey() {
        // 실제로는 더 안전한 키 관리 방식 필요
        String keyString = "MySecretKey12345"; // 16 characters for AES-128
        return new SecretKeySpec(keyString.getBytes(), "AES");
    }

    @SuppressWarnings("unchecked")
    private <T> T convertValue(String value, Class<T> type) {
        if (type == String.class) {
            return (T) value;
        } else if (type == Integer.class || type == int.class) {
            return (T) Integer.valueOf(value);
        } else if (type == Boolean.class || type == boolean.class) {
            return (T) Boolean.valueOf(value);
        } else if (type == Double.class || type == double.class) {
            return (T) Double.valueOf(value);
        } else if (type == Long.class || type == long.class) {
            return (T) Long.valueOf(value);
        }
        throw new IllegalArgumentException("지원하지 않는 타입: " + type);
    }
}
```

## 3. 감사 로그

### 3.1 AuditLog 엔티티
**파일 위치**: `/src/main/java/com/sbs/datasync/entity/AuditLog.java`
```java
package com.sbs.datasync.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "AUDIT_LOG",
       indexes = {
           @Index(name = "IDX_AUDIT_LOG_01", columnList = "ACTION_TYPE"),
           @Index(name = "IDX_AUDIT_LOG_02", columnList = "USERNAME"),
           @Index(name = "IDX_AUDIT_LOG_03", columnList = "ACTION_DATE"),
           @Index(name = "IDX_AUDIT_LOG_04", columnList = "ENTITY_TYPE, ENTITY_ID")
       })
public class AuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "LOG_ID")
    private Long logId;

    @Column(name = "USERNAME", nullable = false, length = 50)
    private String username;

    @Column(name = "ACTION_TYPE", nullable = false, length = 50)
    private String actionType;

    @Column(name = "ENTITY_TYPE", length = 100)
    private String entityType;

    @Column(name = "ENTITY_ID", length = 50)
    private String entityId;

    @Lob
    @Column(name = "OLD_VALUE", columnDefinition = "CLOB")
    private String oldValue;

    @Lob
    @Column(name = "NEW_VALUE", columnDefinition = "CLOB")
    private String newValue;

    @Column(name = "IP_ADDRESS", length = 45)
    private String ipAddress;

    @Column(name = "USER_AGENT", length = 500)
    private String userAgent;

    @Column(name = "ACTION_DATE", nullable = false)
    private LocalDateTime actionDate = LocalDateTime.now();

    @Column(name = "SUCCESS", nullable = false)
    private Boolean success = true;

    @Column(name = "ERROR_MESSAGE", length = 1000)
    private String errorMessage;

    // Constructors, getters, setters...

    public enum ActionType {
        CREATE, READ, UPDATE, DELETE, LOGIN, LOGOUT, BATCH_EXECUTE, BATCH_STOP, CONFIG_CHANGE
    }
}
```

### 3.2 AuditLogService.java
**파일 위치**: `/src/main/java/com/sbs/datasync/service/AuditLogService.java`
```java
package com.sbs.datasync.service;

import com.sbs.datasync.entity.AuditLog;
import com.sbs.datasync.repository.AuditLogRepository;
import org.springframework.scheduling.annotation.Async;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Service;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;

@Service
public class AuditLogService {

    private final AuditLogRepository auditLogRepository;

    public AuditLogService(AuditLogRepository auditLogRepository) {
        this.auditLogRepository = auditLogRepository;
    }

    @Async
    public void logAction(String actionType, String entityType, String entityId,
                         String oldValue, String newValue) {
        try {
            AuditLog log = new AuditLog();
            log.setActionType(actionType);
            log.setEntityType(entityType);
            log.setEntityId(entityId);
            log.setOldValue(oldValue);
            log.setNewValue(newValue);

            // 사용자 정보 설정
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth != null && auth.isAuthenticated()) {
                log.setUsername(auth.getName());
            } else {
                log.setUsername("anonymous");
            }

            // HTTP 요청 정보 설정
            ServletRequestAttributes attributes =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (attributes != null) {
                HttpServletRequest request = attributes.getRequest();
                log.setIpAddress(getClientIpAddress(request));
                log.setUserAgent(request.getHeader("User-Agent"));
            }

            auditLogRepository.save(log);

        } catch (Exception e) {
            logger.error("감사 로그 저장 실패", e);
            // 감사 로그 실패가 메인 비즈니스 로직에 영향을 주지 않도록 예외를 삼킴
        }
    }

    @Async
    public void logBatchExecution(Long jobConfigId, String jobName, String executedBy, boolean success, String errorMessage) {
        AuditLog log = new AuditLog();
        log.setActionType("BATCH_EXECUTE");
        log.setEntityType("BatchJobConfig");
        log.setEntityId(jobConfigId.toString());
        log.setUsername(executedBy);
        log.setNewValue(String.format("Job: %s, Success: %s", jobName, success));
        log.setSuccess(success);
        log.setErrorMessage(errorMessage);

        auditLogRepository.save(log);
    }

    @Async
    public void logLogin(String username, String ipAddress, boolean success, String errorMessage) {
        AuditLog log = new AuditLog();
        log.setActionType("LOGIN");
        log.setUsername(username);
        log.setIpAddress(ipAddress);
        log.setSuccess(success);
        log.setErrorMessage(errorMessage);

        auditLogRepository.save(log);
    }

    private String getClientIpAddress(HttpServletRequest request) {
        String[] headers = {
            "X-Forwarded-For",
            "X-Real-IP",
            "Proxy-Client-IP",
            "WL-Proxy-Client-IP",
            "HTTP_X_FORWARDED_FOR",
            "HTTP_X_FORWARDED",
            "HTTP_X_CLUSTER_CLIENT_IP",
            "HTTP_CLIENT_IP",
            "HTTP_FORWARDED_FOR",
            "HTTP_FORWARDED",
            "HTTP_VIA",
            "REMOTE_ADDR"
        };

        for (String header : headers) {
            String ip = request.getHeader(header);
            if (ip != null && ip.length() != 0 && !"unknown".equalsIgnoreCase(ip)) {
                return ip.split(",")[0];
            }
        }

        return request.getRemoteAddr();
    }
}
```

## 4. 프론트엔드 인증

### 4.1 Login.tsx
**파일 위치**: `/frontend/src/pages/Login.tsx`
```tsx
import React, { useState } from 'react';
import {
  Container,
  Paper,
  TextField,
  Button,
  Typography,
  Alert,
  Box,
  IconButton,
  InputAdornment,
} from '@mui/material';
import { Visibility, VisibilityOff, Security } from '@mui/icons-material';
import { useForm } from 'react-hook-form';
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';
import { useMutation } from '@tanstack/react-query';
import { useNavigate } from 'react-router-dom';

import { authService } from '@/services/authService';
import { useAuthStore } from '@/stores/authStore';

const schema = yup.object({
  username: yup.string().required('사용자명을 입력하세요'),
  password: yup.string().required('비밀번호를 입력하세요'),
});

interface LoginForm {
  username: string;
  password: string;
}

export default function Login() {
  const [showPassword, setShowPassword] = useState(false);
  const navigate = useNavigate();
  const { login } = useAuthStore();

  const { register, handleSubmit, formState: { errors } } = useForm<LoginForm>({
    resolver: yupResolver(schema),
  });

  const loginMutation = useMutation({
    mutationFn: authService.login,
    onSuccess: (data) => {
      login(data.token, data.user);
      navigate('/');
    },
    onError: (error: any) => {
      console.error('로그인 실패:', error);
    },
  });

  const onSubmit = (data: LoginForm) => {
    loginMutation.mutate(data);
  };

  return (
    <Container component="main" maxWidth="sm">
      <Box
        sx={{
          marginTop: 8,
          display: 'flex',
          flexDirection: 'column',
          alignItems: 'center',
        }}
      >
        <Paper elevation={3} sx={{ padding: 4, width: '100%' }}>
          <Box
            sx={{
              display: 'flex',
              flexDirection: 'column',
              alignItems: 'center',
            }}
          >
            <Security sx={{ fontSize: 48, color: 'primary.main', mb: 2 }} />
            <Typography component="h1" variant="h4" gutterBottom>
              SBS DataSync
            </Typography>
            <Typography variant="body2" color="textSecondary" gutterBottom>
              배치 데이터 동기화 시스템
            </Typography>

            {loginMutation.error && (
              <Alert severity="error" sx={{ width: '100%', mt: 2 }}>
                {loginMutation.error.response?.data?.message || '로그인에 실패했습니다.'}
              </Alert>
            )}

            <Box component="form" onSubmit={handleSubmit(onSubmit)} sx={{ mt: 3, width: '100%' }}>
              <TextField
                {...register('username')}
                margin="normal"
                required
                fullWidth
                label="사용자명"
                autoComplete="username"
                autoFocus
                error={!!errors.username}
                helperText={errors.username?.message}
              />
              <TextField
                {...register('password')}
                margin="normal"
                required
                fullWidth
                label="비밀번호"
                type={showPassword ? 'text' : 'password'}
                autoComplete="current-password"
                error={!!errors.password}
                helperText={errors.password?.message}
                InputProps={{
                  endAdornment: (
                    <InputAdornment position="end">
                      <IconButton
                        onClick={() => setShowPassword(!showPassword)}
                        edge="end"
                      >
                        {showPassword ? <VisibilityOff /> : <Visibility />}
                      </IconButton>
                    </InputAdornment>
                  ),
                }}
              />
              <Button
                type="submit"
                fullWidth
                variant="contained"
                sx={{ mt: 3, mb: 2 }}
                disabled={loginMutation.isPending}
              >
                {loginMutation.isPending ? '로그인 중...' : '로그인'}
              </Button>
            </Box>
          </Box>
        </Paper>
      </Box>
    </Container>
  );
}
```

**이유**:
- JWT 토큰 기반 인증 시스템
- 역할 기반 접근 제어 (RBAC)
- 민감한 설정값 암호화
- 상세한 감사 로그 기록
- 사용자 친화적 로그인 UI

이상으로 Phase 7의 보안 및 운영 기능 구현 코드를 제시했습니다.