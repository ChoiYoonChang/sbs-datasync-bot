# Phase 1: 프로젝트 초기 설정 - 구현 코드

## 개요
이 Phase에서는 Spring Boot + React 프로젝트의 기본 구조를 설정하고 DB2 데이터베이스 연결을 구성합니다.

## 1. 프로젝트 구조 생성

### 1.1 build.gradle
**파일 위치**: `/build.gradle`
```gradle
plugins {
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'java'
}

group = 'com.sbs'
version = '1.0.0'
sourceCompatibility = '17'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot Starters
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    // DB2 Driver
    implementation 'com.ibm.db2:jcc:11.5.8.0'

    // JSON 처리
    implementation 'com.fasterxml.jackson.core:jackson-databind'

    // API 문서화
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.2.0'

    // 개발 도구
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'

    // 테스트
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.batch:spring-batch-test'
    testImplementation 'com.h2database:h2'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

**이유**:
- Spring Boot 3.2.0은 안정성과 최신 기능을 제공
- Spring Batch 5.x는 Spring Boot 3와 완벽 호환
- DB2 11.5.8.0은 최신 안정 버전으로 성능 최적화
- OpenAPI는 자동 API 문서 생성으로 개발 효율성 증대

### 1.2 SbsDataSyncBatchApplication.java
**파일 위치**: `/src/main/java/com/sbs/datasync/SbsDataSyncBatchApplication.java`
```java
package com.sbs.datasync;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@SpringBootApplication
@EnableJpaAuditing
public class SbsDataSyncBatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(SbsDataSyncBatchApplication.class, args);
    }
}
```

**이유**:
- @EnableJpaAuditing으로 엔티티 생성/수정 시간 자동 관리
- 단순하고 명확한 메인 클래스로 가독성 확보

## 2. 설정 파일

### 2.1 application.yml
**파일 위치**: `/src/main/resources/application.yml`
```yaml
spring:
  application:
    name: sbs-datasync-batch
  profiles:
    active: dev

server:
  port: 8080
  servlet:
    context-path: /

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,beans
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    com.sbs.datasync: DEBUG
    org.springframework.batch: INFO
    org.springframework.web: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/sbs-datasync-batch.log

springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
```

**이유**:
- 개발/운영 환경 분리로 설정 관리 효율화
- Actuator로 운영 모니터링 기능 제공
- 상세한 로깅으로 디버깅 및 운영 지원
- Swagger UI로 API 문서 자동 생성

### 2.2 application-dev.yml
**파일 위치**: `/src/main/resources/application-dev.yml`
```yaml
spring:
  datasource:
    primary:
      driver-class-name: com.ibm.db2.jcc.DB2Driver
      url: jdbc:db2://localhost:50000/DEVDB
      username: ${DB_DEV_USERNAME:dev_user}
      password: ${DB_DEV_PASSWORD:dev_password}
      hikari:
        maximum-pool-size: 10
        minimum-idle: 2
        idle-timeout: 300000
        connection-timeout: 30000
        validation-timeout: 5000
        leak-detection-threshold: 60000

    secondary:
      driver-class-name: com.ibm.db2.jcc.DB2Driver
      url: jdbc:db2://localhost:50000/TESTDB
      username: ${DB_TEST_USERNAME:test_user}
      password: ${DB_TEST_PASSWORD:test_password}
      hikari:
        maximum-pool-size: 5
        minimum-idle: 1
        idle-timeout: 300000
        connection-timeout: 30000

    source:
      driver-class-name: com.ibm.db2.jcc.DB2Driver
      url: jdbc:db2://localhost:50000/SOURCEDB
      username: ${DB_SOURCE_USERNAME:source_user}
      password: ${DB_SOURCE_PASSWORD:source_password}
      hikari:
        maximum-pool-size: 15
        minimum-idle: 3
        idle-timeout: 300000
        connection-timeout: 30000
        read-only: true

  jpa:
    database: DB2
    show-sql: true
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.DB2Dialect
        format_sql: true
        use_sql_comments: true

  batch:
    job:
      enabled: false
    jdbc:
      initialize-schema: always
      table-prefix: BATCH_
```

**이유**:
- 3개의 데이터소스로 운영/개발/소스 DB 분리
- HikariCP 최적 설정으로 성능 극대화
- 환경변수로 민감 정보 보호
- Hibernate 설정으로 SQL 가시성 및 성능 최적화

### 2.3 application-prod.yml
**파일 위치**: `/src/main/resources/application-prod.yml`
```yaml
spring:
  datasource:
    primary:
      driver-class-name: com.ibm.db2.jcc.DB2Driver
      url: jdbc:db2://${DB_PRIMARY_HOST}:${DB_PRIMARY_PORT:50000}/${DB_PRIMARY_NAME}
      username: ${DB_PRIMARY_USERNAME}
      password: ${DB_PRIMARY_PASSWORD}
      hikari:
        maximum-pool-size: 20
        minimum-idle: 5
        idle-timeout: 600000
        connection-timeout: 30000
        validation-timeout: 5000
        leak-detection-threshold: 60000

    secondary:
      driver-class-name: com.ibm.db2.jcc.DB2Driver
      url: jdbc:db2://${DB_SECONDARY_HOST}:${DB_SECONDARY_PORT:50000}/${DB_SECONDARY_NAME}
      username: ${DB_SECONDARY_USERNAME}
      password: ${DB_SECONDARY_PASSWORD}
      hikari:
        maximum-pool-size: 10
        minimum-idle: 2
        idle-timeout: 600000
        connection-timeout: 30000

    source:
      driver-class-name: com.ibm.db2.jcc.DB2Driver
      url: jdbc:db2://${DB_SOURCE_HOST}:${DB_SOURCE_PORT:50000}/${DB_SOURCE_NAME}
      username: ${DB_SOURCE_USERNAME}
      password: ${DB_SOURCE_PASSWORD}
      hikari:
        maximum-pool-size: 25
        minimum-idle: 5
        idle-timeout: 600000
        connection-timeout: 30000
        read-only: true

  jpa:
    database: DB2
    show-sql: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.DB2Dialect

  batch:
    job:
      enabled: true
    jdbc:
      initialize-schema: never

logging:
  level:
    com.sbs.datasync: INFO
    org.springframework.batch: WARN
    root: WARN
```

**이유**:
- 운영환경 최적 설정으로 성능과 안정성 확보
- 모든 DB 정보를 환경변수로 외부화하여 보안 강화
- 로그 레벨 최적화로 성능 부하 최소화

## 3. 데이터베이스 설정

### 3.1 DatabaseConfig.java
**파일 위치**: `/src/main/java/com/sbs/datasync/config/DatabaseConfig.java`
```java
package com.sbs.datasync.config;

import com.zaxxer.hikari.HikariDataSource;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;

import javax.sql.DataSource;
import java.util.Properties;

@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
    basePackages = "com.sbs.datasync.repository",
    entityManagerFactoryRef = "primaryEntityManagerFactory",
    transactionManagerRef = "primaryTransactionManager"
)
public class DatabaseConfig {

    @Primary
    @Bean(name = "primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create()
                .type(HikariDataSource.class)
                .build();
    }

    @Bean(name = "secondaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create()
                .type(HikariDataSource.class)
                .build();
    }

    @Bean(name = "sourceDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.source")
    public DataSource sourceDataSource() {
        return DataSourceBuilder.create()
                .type(HikariDataSource.class)
                .build();
    }

    @Primary
    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(primaryDataSource());
        em.setPackagesToScan("com.sbs.datasync.entity");

        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);

        Properties properties = new Properties();
        properties.setProperty("hibernate.dialect", "org.hibernate.dialect.DB2Dialect");
        properties.setProperty("hibernate.hbm2ddl.auto", "validate");
        properties.setProperty("hibernate.show_sql", "false");
        properties.setProperty("hibernate.format_sql", "true");
        properties.setProperty("hibernate.use_sql_comments", "true");
        em.setJpaProperties(properties);

        return em;
    }

    @Primary
    @Bean(name = "primaryTransactionManager")
    public PlatformTransactionManager primaryTransactionManager() {
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(
            primaryEntityManagerFactory().getObject()
        );
        return transactionManager;
    }
}
```

**이유**:
- 3개 데이터소스 분리로 역할별 DB 접근 제어
- HikariCP 사용으로 최고 성능의 커넥션 풀 활용
- JPA 설정을 코드로 명시하여 환경별 차이 최소화

### 3.2 JpaAuditingConfig.java
**파일 위치**: `/src/main/java/com/sbs/datasync/config/JpaAuditingConfig.java`
```java
package com.sbs.datasync.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.AuditorAware;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

import java.util.Optional;

@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return new AuditorAwareImpl();
    }

    public static class AuditorAwareImpl implements AuditorAware<String> {
        @Override
        public Optional<String> getCurrentAuditor() {
            Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

            if (authentication == null || !authentication.isAuthenticated()) {
                return Optional.of("system");
            }

            return Optional.of(authentication.getName());
        }
    }
}
```

**이유**:
- JPA Auditing으로 생성자/수정자 정보 자동 관리
- Spring Security와 연동하여 실제 사용자 정보 기록
- 시스템 사용자 기본값으로 안정성 확보

## 4. 기본 엔티티

### 4.1 BaseEntity.java
**파일 위치**: `/src/main/java/com/sbs/datasync/entity/BaseEntity.java`
```java
package com.sbs.datasync.entity;

import jakarta.persistence.Column;
import jakarta.persistence.EntityListeners;
import jakarta.persistence.MappedSuperclass;
import org.springframework.data.annotation.CreatedBy;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedBy;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedBy
    @Column(name = "CREATED_BY", length = 50, updatable = false)
    private String createdBy;

    @CreatedDate
    @Column(name = "CREATED_DATE", updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedBy
    @Column(name = "MODIFIED_BY", length = 50)
    private String modifiedBy;

    @LastModifiedDate
    @Column(name = "MODIFIED_DATE")
    private LocalDateTime modifiedDate;

    // Getters and Setters
    public String getCreatedBy() {
        return createdBy;
    }

    public void setCreatedBy(String createdBy) {
        this.createdBy = createdBy;
    }

    public LocalDateTime getCreatedDate() {
        return createdDate;
    }

    public void setCreatedDate(LocalDateTime createdDate) {
        this.createdDate = createdDate;
    }

    public String getModifiedBy() {
        return modifiedBy;
    }

    public void setModifiedBy(String modifiedBy) {
        this.modifiedBy = modifiedBy;
    }

    public LocalDateTime getModifiedDate() {
        return modifiedDate;
    }

    public void setModifiedDate(LocalDateTime modifiedDate) {
        this.modifiedDate = modifiedDate;
    }
}
```

**이유**:
- 공통 감사 필드를 상속으로 중복 제거
- DB2 컬럼명 규약에 맞는 언더스코어 네이밍
- updatable=false로 생성 정보 보호

### 4.2 BatchJobConfig.java
**파일 위치**: `/src/main/java/com/sbs/datasync/entity/BatchJobConfig.java`
```java
package com.sbs.datasync.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

import java.time.LocalDate;

@Entity
@Table(name = "BATCH_JOB_CONFIG",
       indexes = {
           @Index(name = "IDX_BATCH_JOB_CONFIG_01", columnList = "JOB_NAME"),
           @Index(name = "IDX_BATCH_JOB_CONFIG_02", columnList = "SCHEMA_NAME, TABLE_NAME"),
           @Index(name = "IDX_BATCH_JOB_CONFIG_03", columnList = "IS_ACTIVE")
       })
public class BatchJobConfig extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "JOB_CONFIG_ID")
    private Long jobConfigId;

    @NotBlank(message = "작업명은 필수입니다")
    @Size(max = 100, message = "작업명은 100자를 초과할 수 없습니다")
    @Column(name = "JOB_NAME", nullable = false, length = 100)
    private String jobName;

    @NotBlank(message = "스키마명은 필수입니다")
    @Size(max = 50, message = "스키마명은 50자를 초과할 수 없습니다")
    @Column(name = "SCHEMA_NAME", nullable = false, length = 50)
    private String schemaName = "FAS";

    @NotBlank(message = "테이블명은 필수입니다")
    @Size(max = 100, message = "테이블명은 100자를 초과할 수 없습니다")
    @Column(name = "TABLE_NAME", nullable = false, length = 100)
    private String tableName;

    @Column(name = "START_DATE")
    private LocalDate startDate;

    @Column(name = "END_DATE")
    private LocalDate endDate;

    @Lob
    @Column(name = "INSERT_QUERY", columnDefinition = "CLOB")
    private String insertQuery;

    @Size(max = 500, message = "메모는 500자를 초과할 수 없습니다")
    @Column(name = "MEMO", length = 500)
    private String memo;

    @NotNull
    @Column(name = "IS_ACTIVE", nullable = false, length = 1)
    private String isActive = "Y";

    // Constructors
    public BatchJobConfig() {}

    public BatchJobConfig(String jobName, String schemaName, String tableName) {
        this.jobName = jobName;
        this.schemaName = schemaName;
        this.tableName = tableName;
    }

    // Getters and Setters
    public Long getJobConfigId() {
        return jobConfigId;
    }

    public void setJobConfigId(Long jobConfigId) {
        this.jobConfigId = jobConfigId;
    }

    public String getJobName() {
        return jobName;
    }

    public void setJobName(String jobName) {
        this.jobName = jobName;
    }

    public String getSchemaName() {
        return schemaName;
    }

    public void setSchemaName(String schemaName) {
        this.schemaName = schemaName;
    }

    public String getTableName() {
        return tableName;
    }

    public void setTableName(String tableName) {
        this.tableName = tableName;
    }

    public LocalDate getStartDate() {
        return startDate;
    }

    public void setStartDate(LocalDate startDate) {
        this.startDate = startDate;
    }

    public LocalDate getEndDate() {
        return endDate;
    }

    public void setEndDate(LocalDate endDate) {
        this.endDate = endDate;
    }

    public String getInsertQuery() {
        return insertQuery;
    }

    public void setInsertQuery(String insertQuery) {
        this.insertQuery = insertQuery;
    }

    public String getMemo() {
        return memo;
    }

    public void setMemo(String memo) {
        this.memo = memo;
    }

    public String getIsActive() {
        return isActive;
    }

    public void setIsActive(String isActive) {
        this.isActive = isActive;
    }

    // Business methods
    public boolean isActive() {
        return "Y".equals(this.isActive);
    }

    public void activate() {
        this.isActive = "Y";
    }

    public void deactivate() {
        this.isActive = "N";
    }

    @Override
    public String toString() {
        return "BatchJobConfig{" +
                "jobConfigId=" + jobConfigId +
                ", jobName='" + jobName + '\'' +
                ", schemaName='" + schemaName + '\'' +
                ", tableName='" + tableName + '\'' +
                ", isActive='" + isActive + '\'' +
                '}';
    }
}
```

**이유**:
- Bean Validation으로 데이터 무결성 보장
- DB2 컬럼 타입에 최적화된 매핑
- 인덱스 설정으로 조회 성능 최적화
- 비즈니스 메서드로 가독성 향상

## 5. 테이블 생성 스크립트

### 5.1 schema.sql
**파일 위치**: `/src/main/resources/schema.sql`
```sql
-- 배치 작업 설정 테이블
CREATE TABLE BATCH_JOB_CONFIG (
    JOB_CONFIG_ID       BIGINT NOT NULL GENERATED BY DEFAULT AS IDENTITY,
    JOB_NAME           VARCHAR(100) NOT NULL,
    SCHEMA_NAME        VARCHAR(50) NOT NULL DEFAULT 'FAS',
    TABLE_NAME         VARCHAR(100) NOT NULL,
    START_DATE         DATE,
    END_DATE           DATE,
    INSERT_QUERY       CLOB,
    MEMO               VARCHAR(500),
    IS_ACTIVE          CHAR(1) NOT NULL DEFAULT 'Y',
    CREATED_BY         VARCHAR(50) NOT NULL,
    CREATED_DATE       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    MODIFIED_BY        VARCHAR(50),
    MODIFIED_DATE      TIMESTAMP,
    CONSTRAINT PK_BATCH_JOB_CONFIG PRIMARY KEY (JOB_CONFIG_ID),
    CONSTRAINT CHK_IS_ACTIVE CHECK (IS_ACTIVE IN ('Y', 'N')),
    CONSTRAINT UQ_JOB_NAME UNIQUE (JOB_NAME)
);

-- 인덱스 생성
CREATE INDEX IDX_BATCH_JOB_CONFIG_01 ON BATCH_JOB_CONFIG(JOB_NAME);
CREATE INDEX IDX_BATCH_JOB_CONFIG_02 ON BATCH_JOB_CONFIG(SCHEMA_NAME, TABLE_NAME);
CREATE INDEX IDX_BATCH_JOB_CONFIG_03 ON BATCH_JOB_CONFIG(IS_ACTIVE);

-- 코멘트 추가
COMMENT ON TABLE BATCH_JOB_CONFIG IS '배치 작업 설정 테이블';
COMMENT ON COLUMN BATCH_JOB_CONFIG.JOB_CONFIG_ID IS '작업 설정 ID (PK)';
COMMENT ON COLUMN BATCH_JOB_CONFIG.JOB_NAME IS '작업명';
COMMENT ON COLUMN BATCH_JOB_CONFIG.SCHEMA_NAME IS '스키마명 (기본값: FAS)';
COMMENT ON COLUMN BATCH_JOB_CONFIG.TABLE_NAME IS '테이블명';
COMMENT ON COLUMN BATCH_JOB_CONFIG.START_DATE IS '시작일';
COMMENT ON COLUMN BATCH_JOB_CONFIG.END_DATE IS '종료일';
COMMENT ON COLUMN BATCH_JOB_CONFIG.INSERT_QUERY IS '삽입 쿼리';
COMMENT ON COLUMN BATCH_JOB_CONFIG.MEMO IS '메모';
COMMENT ON COLUMN BATCH_JOB_CONFIG.IS_ACTIVE IS '활성 여부 (Y/N)';
```

**이유**:
- DB2 문법에 맞는 DDL 작성
- 제약조건으로 데이터 무결성 보장
- 인덱스로 조회 성능 최적화
- 코멘트로 테이블 및 컬럼 의미 명시

## 6. 환경 파일

### 6.1 .env.example
**파일 위치**: `/.env.example`
```bash
# Development Database
DB_DEV_USERNAME=dev_user
DB_DEV_PASSWORD=dev_password

# Test Database
DB_TEST_USERNAME=test_user
DB_TEST_PASSWORD=test_password

# Source Database (Read-Only)
DB_SOURCE_USERNAME=source_readonly_user
DB_SOURCE_PASSWORD=source_readonly_password

# Production Primary Database
DB_PRIMARY_HOST=prod-primary-db.company.com
DB_PRIMARY_PORT=50000
DB_PRIMARY_NAME=PRIMARYDB
DB_PRIMARY_USERNAME=primary_user
DB_PRIMARY_PASSWORD=primary_password

# Production Secondary Database
DB_SECONDARY_HOST=prod-secondary-db.company.com
DB_SECONDARY_PORT=50000
DB_SECONDARY_NAME=SECONDARYDB
DB_SECONDARY_USERNAME=secondary_user
DB_SECONDARY_PASSWORD=secondary_password

# Production Source Database
DB_SOURCE_HOST=prod-source-db.company.com
DB_SOURCE_PORT=50000
DB_SOURCE_NAME=SOURCEDB
DB_SOURCE_USERNAME=source_readonly_user
DB_SOURCE_PASSWORD=source_readonly_password
```

**이유**:
- 환경변수로 민감 정보 보호
- 개발팀 간 환경 설정 표준화
- 운영 환경 배포 시 보안 강화

### 6.2 .gitignore
**파일 위치**: `/.gitignore`
```gitignore
# Java
*.class
*.jar
*.war
*.ear
*.zip
*.tar.gz
*.rar

# Gradle
.gradle
build/
!gradle/wrapper/gradle-wrapper.jar
!gradle/wrapper/gradle-wrapper.properties

# IDE
.idea/
*.iws
*.iml
*.ipr
out/
.vscode/

# OS
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# Logs
logs/
*.log

# Database
*.db
*.sqlite

# Environment variables
.env
.env.local
.env.development
.env.test
.env.production

# Application specific
batch-jobs/
temp/

# Node.js (frontend)
frontend/node_modules/
frontend/dist/
frontend/.env.local
frontend/.env.development.local
frontend/.env.test.local
frontend/.env.production.local

# npm
npm-debug.log*
yarn-debug.log*
yarn-error.log*
```

**이유**:
- 빌드 파일과 임시 파일 제외로 저장소 정리
- 민감 정보 파일 제외로 보안 강화
- IDE별 설정 파일 제외로 개발 환경 독립성 보장

## 7. React 프론트엔드 기본 설정

### 7.1 package.json (프론트엔드)
**파일 위치**: `/frontend/package.json`
```json
{
  "name": "sbs-datasync-frontend",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.15.0",
    "axios": "^1.5.0",
    "@tanstack/react-query": "^4.35.0",
    "zustand": "^4.4.0",
    "date-fns": "^2.30.0",
    "@emotion/react": "^11.11.0",
    "@emotion/styled": "^11.11.0",
    "@mui/material": "^5.14.0",
    "@mui/icons-material": "^5.14.0",
    "@mui/x-date-pickers": "^6.10.0",
    "@mui/x-data-grid": "^6.10.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.15",
    "@types/react-dom": "^18.2.7",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "@vitejs/plugin-react": "^4.0.3",
    "eslint": "^8.45.0",
    "eslint-plugin-react-hooks": "^4.6.0",
    "eslint-plugin-react-refresh": "^0.4.3",
    "typescript": "^5.0.2",
    "vite": "^4.4.5"
  }
}
```

**이유**:
- Material-UI로 일관성 있는 디자인 시스템 구축
- React Query로 서버 상태 관리 최적화
- Zustand로 클라이언트 상태 관리 단순화
- TypeScript로 타입 안정성 확보

### 7.2 vite.config.ts
**파일 위치**: `/frontend/vite.config.ts`
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        secure: false,
      }
    }
  },
  build: {
    outDir: '../src/main/resources/static',
    emptyOutDir: true
  }
})
```

**이유**:
- 백엔드 API 프록시 설정으로 CORS 문제 해결
- 빌드 결과를 Spring Boot static 폴더에 배치하여 단일 배포
- 개발 시 Hot Reload 지원

이상으로 Phase 1의 모든 구현 코드를 제시했습니다. 이 코드들을 순서대로 구현하시면 Spring Boot + React 기반의 기본 프로젝트 구조가 완성됩니다.