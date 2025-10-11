# Employee Microservice - Technical Design Document

## Executive Summary

This document outlines the technical architecture for migrating legacy mainframe employee management functionality to a modern Spring Boot microservice deployed on Azure Kubernetes Service (AKS). The solution leverages Spring Boot 3.2, JDK 21, SQL Server backend, Spring Batch for data processing, and Spring Data Envers for comprehensive auditing.

## Architecture Overview

### Technology Stack
- **Framework**: Spring Boot 3.2.0
- **Java Version**: JDK 21
- **Database**: Microsoft SQL Server 2019+
- **ORM**: Spring Data JPA with Hibernate
- **Auditing**: Spring Data Envers
- **Batch Processing**: Spring Batch 5.0
- **Containerization**: Docker with JIB plugin
- **Orchestration**: Azure Kubernetes Service (AKS)
- **CI/CD**: GitHub Actions
- **Build Tool**: Maven 3.9.x

### Microservice Architecture Principles
- **Single Responsibility**: Employee domain management
- **Database per Service**: Dedicated SQL Server database
- **API-First Design**: RESTful APIs with OpenAPI 3.0 specification
- **Event-Driven Architecture**: Domain events for state changes
- **Observability**: Built-in monitoring and logging

## Database Design

### Core Employee Table
```sql
-- DDL for Employee table
CREATE TABLE employee (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    employee_number VARCHAR(20) UNIQUE NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone_number VARCHAR(20),
    hire_date DATE NOT NULL,
    job_title VARCHAR(100),
    department VARCHAR(100),
    salary DECIMAL(12,2),
    manager_id BIGINT,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    created_date DATETIME2(7) NOT NULL,
    created_by VARCHAR(100) NOT NULL,
    last_modified_date DATETIME2(7),
    last_modified_by VARCHAR(100),
    version BIGINT DEFAULT 0,
    CONSTRAINT FK_employee_manager FOREIGN KEY (manager_id) REFERENCES employee(id),
    CONSTRAINT CK_employee_status CHECK (status IN ('ACTIVE', 'INACTIVE', 'TERMINATED'))
);

-- Indexes for performance
CREATE INDEX IDX_employee_number ON employee(employee_number);
CREATE INDEX IDX_employee_email ON employee(email);
CREATE INDEX IDX_employee_department ON employee(department);
CREATE INDEX IDX_employee_manager ON employee(manager_id);
CREATE INDEX IDX_employee_status ON employee(status);
```

### Audit Tables (Created by Envers)
```sql
-- Envers will auto-create these tables
-- employee_aud - stores all historical versions
-- revinfo - stores revision information with timestamps
-- Custom revision entity for enhanced auditing

CREATE TABLE custom_revinfo (
    rev INT IDENTITY(1,1) PRIMARY KEY,
    revtstmp BIGINT,
    username VARCHAR(100),
    user_role VARCHAR(50),
    ip_address VARCHAR(45),
    operation_type VARCHAR(20)
);

-- employee_aud table structure (auto-created by Envers)
-- This mirrors employee table structure with additional columns:
-- rev (references custom_revinfo.rev)
-- revtype (0=INSERT, 1=UPDATE, 2=DELETE)
```

### Sample Data (DML)
```sql
-- Sample employee data
INSERT INTO employee (employee_number, first_name, last_name, email, phone_number, 
                     hire_date, job_title, department, salary, status, 
                     created_date, created_by, last_modified_date, last_modified_by)
VALUES 
('EMP001', 'John', 'Doe', 'john.doe@company.com', '+1-555-0101', 
 '2023-01-15', 'Senior Developer', 'Engineering', 95000.00, 'ACTIVE',
 GETDATE(), 'system', GETDATE(), 'system'),
 
('EMP002', 'Jane', 'Smith', 'jane.smith@company.com', '+1-555-0102', 
 '2023-02-01', 'Product Manager', 'Product', 110000.00, 'ACTIVE',
 GETDATE(), 'system', GETDATE(), 'system'),
 
('EMP003', 'Mike', 'Johnson', 'mike.johnson@company.com', '+1-555-0103', 
 '2023-03-01', 'Team Lead', 'Engineering', 120000.00, 'ACTIVE',
 GETDATE(), 'system', GETDATE(), 'system');

-- Set manager relationships
UPDATE employee SET manager_id = (SELECT id FROM employee WHERE employee_number = 'EMP003')
WHERE employee_number = 'EMP001';
```

## Spring Boot Application Structure

### Maven Configuration (pom.xml)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.company.microservices</groupId>
    <artifactId>employee-service</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    <name>employee-service</name>
    <description>Employee Management Microservice</description>
    
    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Starters -->
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
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-batch</artifactId>
        </dependency>
        
        <!-- Database -->
        <dependency>
            <groupId>com.microsoft.sqlserver</groupId>
            <artifactId>mssql-jdbc</artifactId>
            <version>12.6.1.jre11</version>
        </dependency>
        
        <!-- Auditing -->
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-envers</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-envers</artifactId>
        </dependency>
        
        <!-- Documentation -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.2.0</version>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>mssqlserver</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            
            <!-- JIB for containerization -->
            <plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>3.4.0</version>
                <configuration>
                    <from>
                        <image>openjdk:21-jre-slim</image>
                    </from>
                    <to>
                        <image>company-registry.azurecr.io/employee-service</image>
                        <tags>
                            <tag>${project.version}</tag>
                            <tag>latest</tag>
                        </tags>
                    </to>
                    <container>
                        <jvmFlags>
                            <jvmFlag>-Xms512m</jvmFlag>
                            <jvmFlag>-Xmx1024m</jvmFlag>
                        </jvmFlags>
                        <ports>
                            <port>8080</port>
                        </ports>
                    </container>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### Application Configuration
```yaml
# application.yml
server:
  port: 8080
  servlet:
    context-path: /api/v1

spring:
  application:
    name: employee-service
  
  datasource:
    url: jdbc:sqlserver://${DB_HOST:localhost}:${DB_PORT:1433};databaseName=${DB_NAME:employee_db};encrypt=true;trustServerCertificate=true
    username: ${DB_USERNAME:sa}
    password: ${DB_PASSWORD:password}
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
    
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.SQLServerDialect
        format_sql: true
        
  batch:
    initialize-schema: always
    job:
      enabled: false
      
  # Envers Configuration
  jpa:
    properties:
      org:
        hibernate:
          envers:
            audit_table_suffix: _aud
            revision_field_name: rev
            revision_type_field_name: revtype
            store_data_at_delete: true

# Management and Monitoring
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true

# Logging
logging:
  level:
    com.company.microservices: DEBUG
    org.springframework.web: INFO
    org.springframework.security: DEBUG
  pattern:
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

## Core Application Code

### Main Application Class
```java
package com.company.microservices.employee;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.envers.repository.support.EnversRevisionRepositoryFactoryBean;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@SpringBootApplication
@EnableJpaAuditing
@EnableJpaRepositories(repositoryFactoryBeanClass = EnversRevisionRepositoryFactoryBean.class)
public class EmployeeServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(EmployeeServiceApplication.class, args);
    }
}
```

### Employee Entity with Auditing
```java
package com.company.microservices.employee.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import org.hibernate.envers.Audited;
import org.hibernate.envers.NotAudited;
import org.springframework.data.annotation.CreatedBy;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedBy;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "employee")
@Audited
@EntityListeners(AuditingEntityListener.class)
public class Employee {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "employee_number", unique = true, nullable = false, length = 20)
    @NotBlank(message = "Employee number is required")
    @Size(max = 20, message = "Employee number must not exceed 20 characters")
    private String employeeNumber;
    
    @Column(name = "first_name", nullable = false, length = 100)
    @NotBlank(message = "First name is required")
    @Size(max = 100, message = "First name must not exceed 100 characters")
    private String firstName;
    
    @Column(name = "last_name", nullable = false, length = 100)
    @NotBlank(message = "Last name is required")
    @Size(max = 100, message = "Last name must not exceed 100 characters")
    private String lastName;
    
    @Column(name = "email", unique = true, nullable = false)
    @NotBlank(message = "Email is required")
    @Email(message = "Email should be valid")
    private String email;
    
    @Column(name = "phone_number", length = 20)
    @Size(max = 20, message = "Phone number must not exceed 20 characters")
    private String phoneNumber;
    
    @Column(name = "hire_date", nullable = false)
    @NotNull(message = "Hire date is required")
    private LocalDate hireDate;
    
    @Column(name = "job_title", length = 100)
    @Size(max = 100, message = "Job title must not exceed 100 characters")
    private String jobTitle;
    
    @Column(name = "department", length = 100)
    @Size(max = 100, message = "Department must not exceed 100 characters")
    private String department;
    
    @Column(name = "salary", precision = 12, scale = 2)
    @DecimalMin(value = "0.0", inclusive = false, message = "Salary must be positive")
    private BigDecimal salary;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "manager_id")
    private Employee manager;
    
    @OneToMany(mappedBy = "manager", fetch = FetchType.LAZY)
    @NotAudited
    private List<Employee> subordinates = new ArrayList<>();
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status", length = 20)
    private EmployeeStatus status = EmployeeStatus.ACTIVE;
    
    @CreatedDate
    @Column(name = "created_date", nullable = false, updatable = false)
    private LocalDateTime createdDate;
    
    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false, length = 100)
    private String createdBy;
    
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private LocalDateTime lastModifiedDate;
    
    @LastModifiedBy
    @Column(name = "last_modified_by", length = 100)
    private String lastModifiedBy;
    
    @Version
    @Column(name = "version")
    private Long version;
    
    // Constructors
    public Employee() {}
    
    public Employee(String employeeNumber, String firstName, String lastName, String email) {
        this.employeeNumber = employeeNumber;
        this.firstName = firstName;
        this.lastName = lastName;
        this.email = email;
        this.status = EmployeeStatus.ACTIVE;
    }
    
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getEmployeeNumber() { return employeeNumber; }
    public void setEmployeeNumber(String employeeNumber) { this.employeeNumber = employeeNumber; }
    
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    
    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public String getPhoneNumber() { return phoneNumber; }
    public void setPhoneNumber(String phoneNumber) { this.phoneNumber = phoneNumber; }
    
    public LocalDate getHireDate() { return hireDate; }
    public void setHireDate(LocalDate hireDate) { this.hireDate = hireDate; }
    
    public String getJobTitle() { return jobTitle; }
    public void setJobTitle(String jobTitle) { this.jobTitle = jobTitle; }
    
    public String getDepartment() { return department; }
    public void setDepartment(String department) { this.department = department; }
    
    public BigDecimal getSalary() { return salary; }
    public void setSalary(BigDecimal salary) { this.salary = salary; }
    
    public Employee getManager() { return manager; }
    public void setManager(Employee manager) { this.manager = manager; }
    
    public List<Employee> getSubordinates() { return subordinates; }
    public void setSubordinates(List<Employee> subordinates) { this.subordinates = subordinates; }
    
    public EmployeeStatus getStatus() { return status; }
    public void setStatus(EmployeeStatus status) { this.status = status; }
    
    public LocalDateTime getCreatedDate() { return createdDate; }
    public void setCreatedDate(LocalDateTime createdDate) { this.createdDate = createdDate; }
    
    public String getCreatedBy() { return createdBy; }
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
    
    public LocalDateTime getLastModifiedDate() { return lastModifiedDate; }
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) { this.lastModifiedDate = lastModifiedDate; }
    
    public String getLastModifiedBy() { return lastModifiedBy; }
    public void setLastModifiedBy(String lastModifiedBy) { this.lastModifiedBy = lastModifiedBy; }
    
    public Long getVersion() { return version; }
    public void setVersion(Long version) { this.version = version; }
    
    // Business Methods
    public String getFullName() {
        return firstName + " " + lastName;
    }
    
    public void terminate() {
        this.status = EmployeeStatus.TERMINATED;
    }
    
    public void reactivate() {
        this.status = EmployeeStatus.ACTIVE;
    }
    
    public boolean isActive() {
        return EmployeeStatus.ACTIVE.equals(this.status);
    }
}
```

### Employee Status Enum
```java
package com.company.microservices.employee.entity;

public enum EmployeeStatus {
    ACTIVE("Active"),
    INACTIVE("Inactive"), 
    TERMINATED("Terminated");
    
    private final String displayName;
    
    EmployeeStatus(String displayName) {
        this.displayName = displayName;
    }
    
    public String getDisplayName() {
        return displayName;
    }
}
```

### Custom Revision Entity for Enhanced Auditing
```java
package com.company.microservices.employee.audit;

import jakarta.persistence.*;
import org.hibernate.envers.DefaultRevisionEntity;
import org.hibernate.envers.RevisionEntity;

@Entity
@Table(name = "custom_revinfo")
@RevisionEntity(CustomRevisionListener.class)
public class CustomRevisionEntity extends DefaultRevisionEntity {
    
    @Column(name = "username", length = 100)
    private String username;
    
    @Column(name = "user_role", length = 50)
    private String userRole;
    
    @Column(name = "ip_address", length = 45)
    private String ipAddress;
    
    @Column(name = "operation_type", length = 20)
    private String operationType;
    
    // Constructors
    public CustomRevisionEntity() {}
    
    // Getters and Setters
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    
    public String getUserRole() { return userRole; }
    public void setUserRole(String userRole) { this.userRole = userRole; }
    
    public String getIpAddress() { return ipAddress; }
    public void setIpAddress(String ipAddress) { this.ipAddress = ipAddress; }
    
    public String getOperationType() { return operationType; }
    public void setOperationType(String operationType) { this.operationType = operationType; }
}
```

### Custom Revision Listener
```java
package com.company.microservices.employee.audit;

import jakarta.servlet.http.HttpServletRequest;
import org.hibernate.envers.RevisionListener;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

public class CustomRevisionListener implements RevisionListener {
    
    @Override
    public void newRevision(Object revisionEntity) {
        CustomRevisionEntity customRevisionEntity = (CustomRevisionEntity) revisionEntity;
        
        try {
            ServletRequestAttributes requestAttributes = 
                (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            HttpServletRequest request = requestAttributes.getRequest();
            
            // Extract user information from security context or request headers
            String username = extractUsername(request);
            String userRole = extractUserRole(request);
            String ipAddress = getClientIpAddress(request);
            String operationType = request.getMethod();
            
            customRevisionEntity.setUsername(username);
            customRevisionEntity.setUserRole(userRole);
            customRevisionEntity.setIpAddress(ipAddress);
            customRevisionEntity.setOperationType(operationType);
            
        } catch (Exception e) {
            // Fallback values
            customRevisionEntity.setUsername("system");
            customRevisionEntity.setUserRole("SYSTEM");
            customRevisionEntity.setIpAddress("unknown");
            customRevisionEntity.setOperationType("UNKNOWN");
        }
    }
    
    private String extractUsername(HttpServletRequest request) {
        // Extract from security context or custom header
        String username = request.getHeader("X-User-Name");
        return username != null ? username : "anonymous";
    }
    
    private String extractUserRole(HttpServletRequest request) {
        // Extract from security context or custom header
        String role = request.getHeader("X-User-Role");
        return role != null ? role : "USER";
    }
    
    private String getClientIpAddress(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        
        String xRealIP = request.getHeader("X-Real-IP");
        if (xRealIP != null && !xRealIP.isEmpty()) {
            return xRealIP;
        }
        
        return request.getRemoteAddr();
    }
}
```

### Repository Layer
```java
package com.company.microservices.employee.repository;

import com.company.microservices.employee.entity.Employee;
import com.company.microservices.employee.entity.EmployeeStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.data.repository.history.RevisionRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long>, 
                                           RevisionRepository<Employee, Long, Long> {
    
    Optional<Employee> findByEmployeeNumber(String employeeNumber);
    
    Optional<Employee> findByEmail(String email);
    
    List<Employee> findByStatus(EmployeeStatus status);
    
    List<Employee> findByDepartment(String department);
    
    List<Employee> findByManagerId(Long managerId);
    
    @Query("SELECT e FROM Employee e WHERE " +
           "(:firstName IS NULL OR LOWER(e.firstName) LIKE LOWER(CONCAT('%', :firstName, '%'))) AND " +
           "(:lastName IS NULL OR LOWER(e.lastName) LIKE LOWER(CONCAT('%', :lastName, '%'))) AND " +
           "(:department IS NULL OR LOWER(e.department) = LOWER(:department)) AND " +
           "(:status IS NULL OR e.status = :status)")
    Page<Employee> findEmployeesWithFilters(
        @Param("firstName") String firstName,
        @Param("lastName") String lastName,
        @Param("department") String department,
        @Param("status") EmployeeStatus status,
        Pageable pageable
    );
    
    @Query("SELECT DISTINCT e.department FROM Employee e WHERE e.department IS NOT NULL ORDER BY e.department")
    List<String> findAllDepartments();
    
    @Query("SELECT COUNT(e) FROM Employee e WHERE e.manager.id = :managerId AND e.status = 'ACTIVE'")
    Long countActiveSubordinates(@Param("managerId") Long managerId);
    
    boolean existsByEmployeeNumber(String employeeNumber);
    
    boolean existsByEmail(String email);
}
```

### DTO Classes
```java
// EmployeeDTO.java
package com.company.microservices.employee.dto;

import com.company.microservices.employee.entity.EmployeeStatus;
import jakarta.validation.constraints.*;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class EmployeeDTO {
    
    private Long id;
    
    @NotBlank(message = "Employee number is required")
    @Size(max = 20, message = "Employee number must not exceed 20 characters")
    private String employeeNumber;
    
    @NotBlank(message = "First name is required")
    @Size(max = 100, message = "First name must not exceed 100 characters")
    private String firstName;
    
    @NotBlank(message = "Last name is required") 
    @Size(max = 100, message = "Last name must not exceed 100 characters")
    private String lastName;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email should be valid")
    private String email;
    
    @Size(max = 20, message = "Phone number must not exceed 20 characters")
    private String phoneNumber;
    
    @NotNull(message = "Hire date is required")
    private LocalDate hireDate;
    
    @Size(max = 100, message = "Job title must not exceed 100 characters")
    private String jobTitle;
    
    @Size(max = 100, message = "Department must not exceed 100 characters")
    private String department;
    
    @DecimalMin(value = "0.0", inclusive = false, message = "Salary must be positive")
    private BigDecimal salary;
    
    private Long managerId;
    private String managerName;
    private EmployeeStatus status;
    private LocalDateTime createdDate;
    private String createdBy;
    private LocalDateTime lastModifiedDate;
    private String lastModifiedBy;
    private Long version;
    
    // Constructors
    public EmployeeDTO() {}
    
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getEmployeeNumber() { return employeeNumber; }
    public void setEmployeeNumber(String employeeNumber) { this.employeeNumber = employeeNumber; }
    
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    
    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public String getPhoneNumber() { return phoneNumber; }
    public void setPhoneNumber(String phoneNumber) { this.phoneNumber = phoneNumber; }
    
    public LocalDate getHireDate() { return hireDate; }
    public void setHireDate(LocalDate hireDate) { this.hireDate = hireDate; }
    
    public String getJobTitle() { return jobTitle; }
    public void setJobTitle(String jobTitle) { this.jobTitle = jobTitle; }
    
    public String getDepartment() { return department; }
    public void setDepartment(String department) { this.department = department; }
    
    public BigDecimal getSalary() { return salary; }
    public void setSalary(BigDecimal salary) { this.salary = salary; }
    
    public Long getManagerId() { return managerId; }
    public void setManagerId(Long managerId) { this.managerId = managerId; }
    
    public String getManagerName() { return managerName; }
    public void setManagerName(String managerName) { this.managerName = managerName; }
    
    public EmployeeStatus getStatus() { return status; }
    public void setStatus(EmployeeStatus status) { this.status = status; }
    
    public LocalDateTime getCreatedDate() { return createdDate; }
    public void setCreatedDate(LocalDateTime createdDate) { this.createdDate = createdDate; }
    
    public String getCreatedBy() { return createdBy; }
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
    
    public LocalDateTime getLastModifiedDate() { return lastModifiedDate; }
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) { this.lastModifiedDate = lastModifiedDate; }
    
    public String getLastModifiedBy() { return lastModifiedBy; }
    public void setLastModifiedBy(String lastModifiedBy) { this.lastModifiedBy = lastModifiedBy; }
    
    public Long getVersion() { return version; }
    public void setVersion(Long version) { this.version = version; }
    
    public String getFullName() {
        return firstName + " " + lastName;
    }
}
```

### Service Layer
```java
package com.company.microservices.employee.service;

import com.company.microservices.employee.dto.EmployeeDTO;
import com.company.microservices.employee.entity.Employee;
import com.company.microservices.employee.entity.EmployeeStatus;
import com.company.microservices.employee.exception.EmployeeNotFoundException;
import com.company.microservices.employee.exception.DuplicateEmployeeException;
import com.company.microservices.employee.mapper.EmployeeMapper;
import com.company.microservices.employee.repository.EmployeeRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.history.Revision;
import org.springframework.data.history.Revisions;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Service
@Transactional
public class EmployeeService {
    
    private final EmployeeRepository employeeRepository;
    private final EmployeeMapper employeeMapper;
    
    @Autowired
    public EmployeeService(EmployeeRepository employeeRepository, EmployeeMapper employeeMapper) {
        this.employeeRepository = employeeRepository;
        this.employeeMapper = employeeMapper;
    }
    
    @Transactional(readOnly = true)
    public List<EmployeeDTO> getAllEmployees() {
        List<Employee> employees = employeeRepository.findAll();
        return employeeMapper.toEmployeeDTOList(employees);
    }
    
    @Transactional(readOnly = true)
    public Page<EmployeeDTO> getEmployeesWithFilters(String firstName, String lastName, 
                                                   String department, EmployeeStatus status, 
                                                   Pageable pageable) {
        Page<Employee> employees = employeeRepository.findEmployeesWithFilters(
            firstName, lastName, department, status, pageable);
        return employees.map(employeeMapper::toEmployeeDTO);
    }
    
    @Transactional(readOnly = true)
    public EmployeeDTO getEmployeeById(Long id) {
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new EmployeeNotFoundException("Employee not found with id: " + id));
        return employeeMapper.toEmployeeDTO(employee);
    }
    
    @Transactional(readOnly = true)
    public EmployeeDTO getEmployeeByEmployeeNumber(String employeeNumber) {
        Employee employee = employeeRepository.findByEmployeeNumber(employeeNumber)
            .orElseThrow(() -> new EmployeeNotFoundException(
                "Employee not found with employee number: " + employeeNumber));
        return employeeMapper.toEmployeeDTO(employee);
    }
    
    public EmployeeDTO createEmployee(EmployeeDTO employeeDTO) {
        // Check for duplicates
        if (employeeRepository.existsByEmployeeNumber(employeeDTO.getEmployeeNumber())) {
            throw new DuplicateEmployeeException(
                "Employee already exists with employee number: " + employeeDTO.getEmployeeNumber());
        }
        
        if (employeeRepository.existsByEmail(employeeDTO.getEmail())) {
            throw new DuplicateEmployeeException(
                "Employee already exists with email: " + employeeDTO.getEmail());
        }
        
        Employee employee = employeeMapper.toEmployee(employeeDTO);
        
        // Set manager if managerId is provided
        if (employeeDTO.getManagerId() != null) {
            Employee manager = employeeRepository.findById(employeeDTO.getManagerId())
                .orElseThrow(() -> new EmployeeNotFoundException(
                    "Manager not found with id: " + employeeDTO.getManagerId()));
            employee.setManager(manager);
        }
        
        employee.setStatus(EmployeeStatus.ACTIVE);
        Employee savedEmployee = employeeRepository.save(employee);
        return employeeMapper.toEmployeeDTO(savedEmployee);
    }
    
    public EmployeeDTO updateEmployee(Long id, EmployeeDTO employeeDTO) {
        Employee existingEmployee = employeeRepository.findById(id)
            .orElseThrow(() -> new EmployeeNotFoundException("Employee not found with id: " + id));
        
        // Check for email uniqueness if email is being changed
        if (!existingEmployee.getEmail().equals(employeeDTO.getEmail()) && 
            employeeRepository.existsByEmail(employeeDTO.getEmail())) {
            throw new DuplicateEmployeeException(
                "Employee already exists with email: " + employeeDTO.getEmail());
        }
        
        // Update fields
        existingEmployee.setFirstName(employeeDTO.getFirstName());
        existingEmployee.setLastName(employeeDTO.getLastName());
        existingEmployee.setEmail(employeeDTO.getEmail());
        existingEmployee.setPhoneNumber(employeeDTO.getPhoneNumber());
        existingEmployee.setJobTitle(employeeDTO.getJobTitle());
        existingEmployee.setDepartment(employeeDTO.getDepartment());
        existingEmployee.setSalary(employeeDTO.getSalary());
        
        // Update manager if managerId is provided
        if (employeeDTO.getManagerId() != null) {
            if (!employeeDTO.getManagerId().equals(
                existingEmployee.getManager() != null ? existingEmployee.getManager().getId() : null)) {
                Employee manager = employeeRepository.findById(employeeDTO.getManagerId())
                    .orElseThrow(() -> new EmployeeNotFoundException(
                        "Manager not found with id: " + employeeDTO.getManagerId()));
                existingEmployee.setManager(manager);
            }
        } else {
            existingEmployee.setManager(null);
        }
        
        Employee savedEmployee = employeeRepository.save(existingEmployee);
        return employeeMapper.toEmployeeDTO(savedEmployee);
    }
    
    public void deleteEmployee(Long id) {
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new EmployeeNotFoundException("Employee not found with id: " + id));
        
        // Check if employee has subordinates
        Long subordinateCount = employeeRepository.countActiveSubordinates(id);
        if (subordinateCount > 0) {
            throw new IllegalStateException(
                "Cannot delete employee with active subordinates. Please reassign subordinates first.");
        }
        
        employeeRepository.delete(employee);
    }
    
    public EmployeeDTO terminateEmployee(Long id) {
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new EmployeeNotFoundException("Employee not found with id: " + id));
        
        employee.terminate();
        Employee savedEmployee = employeeRepository.save(employee);
        return employeeMapper.toEmployeeDTO(savedEmployee);
    }
    
    public EmployeeDTO reactivateEmployee(Long id) {
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new EmployeeNotFoundException("Employee not found with id: " + id));
        
        employee.reactivate();
        Employee savedEmployee = employeeRepository.save(employee);
        return employeeMapper.toEmployeeDTO(savedEmployee);
    }
    
    @Transactional(readOnly = true)
    public List<String> getAllDepartments() {
        return employeeRepository.findAllDepartments();
    }
    
    @Transactional(readOnly = true)
    public List<EmployeeDTO> getEmployeesByDepartment(String department) {
        List<Employee> employees = employeeRepository.findByDepartment(department);
        return employeeMapper.toEmployeeDTOList(employees);
    }
    
    @Transactional(readOnly = true)
    public List<EmployeeDTO> getSubordinates(Long managerId) {
        List<Employee> subordinates = employeeRepository.findByManagerId(managerId);
        return employeeMapper.toEmployeeDTOList(subordinates);
    }
    
    // Audit-related methods
    @Transactional(readOnly = true)
    public Revisions<Long, Employee> getEmployeeRevisions(Long id) {
        return employeeRepository.findRevisions(id);
    }
    
    @Transactional(readOnly = true)
    public Optional<Revision<Long, Employee>> getEmployeeRevision(Long id, Long revisionNumber) {
        return employeeRepository.findRevision(id, revisionNumber);
    }
    
    @Transactional(readOnly = true)
    public Revision<Long, Employee> getLastEmployeeRevision(Long id) {
        return employeeRepository.findLastChangeRevision(id)
            .orElseThrow(() -> new EmployeeNotFoundException(
                "No revision found for employee with id: " + id));
    }
}
```

### Controller Layer
```java
package com.company.microservices.employee.controller;

import com.company.microservices.employee.dto.EmployeeDTO;
import com.company.microservices.employee.entity.Employee;
import com.company.microservices.employee.entity.EmployeeStatus;
import com.company.microservices.employee.service.EmployeeService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.history.Revision;
import org.springframework.data.history.Revisions;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Optional;

@RestController
@RequestMapping("/employees")
@Tag(name = "Employee Management", description = "CRUD operations for Employee management")
@CrossOrigin(origins = "http://localhost:4200", allowCredentials = "true")
public class EmployeeController {
    
    private final EmployeeService employeeService;
    
    @Autowired
    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }
    
    @GetMapping
    @Operation(summary = "Get all employees", description = "Retrieve all employees with optional filtering and pagination")
    @ApiResponse(responseCode = "200", description = "Successfully retrieved employees")
    public ResponseEntity<Page<EmployeeDTO>> getAllEmployees(
            @Parameter(description = "Filter by first name") @RequestParam(required = false) String firstName,
            @Parameter(description = "Filter by last name") @RequestParam(required = false) String lastName,
            @Parameter(description = "Filter by department") @RequestParam(required = false) String department,
            @Parameter(description = "Filter by status") @RequestParam(required = false) EmployeeStatus status,
            @PageableDefault(size = 20, sort = "lastName", direction = Sort.Direction.ASC) Pageable pageable) {
        
        Page<EmployeeDTO> employees = employeeService.getEmployeesWithFilters(
            firstName, lastName, department, status, pageable);
        return ResponseEntity.ok(employees);
    }
    
    @GetMapping("/{id}")
    @Operation(summary = "Get employee by ID", description = "Retrieve a specific employee by their ID")
    @ApiResponse(responseCode = "200", description = "Successfully retrieved employee")
    @ApiResponse(responseCode = "404", description = "Employee not found")
    public ResponseEntity<EmployeeDTO> getEmployeeById(
            @Parameter(description = "Employee ID") @PathVariable Long id) {
        EmployeeDTO employee = employeeService.getEmployeeById(id);
        return ResponseEntity.ok(employee);
    }
    
    @GetMapping("/by-employee-number/{employeeNumber}")
    @Operation(summary = "Get employee by employee number", 
               description = "Retrieve a specific employee by their employee number")
    @ApiResponse(responseCode = "200", description = "Successfully retrieved employee")
    @ApiResponse(responseCode = "404", description = "Employee not found")
    public ResponseEntity<EmployeeDTO> getEmployeeByEmployeeNumber(
            @Parameter(description = "Employee Number") @PathVariable String employeeNumber) {
        EmployeeDTO employee = employeeService.getEmployeeByEmployeeNumber(employeeNumber);
        return ResponseEntity.ok(employee);
    }
    
    @PostMapping
    @Operation(summary = "Create new employee", description = "Create a new employee record")
    @ApiResponse(responseCode = "201", description = "Employee successfully created")
    @ApiResponse(responseCode = "400", description = "Invalid input data")
    @ApiResponse(responseCode = "409", description = "Employee already exists")
    public ResponseEntity<EmployeeDTO> createEmployee(
            @Parameter(description = "Employee data") @Valid @RequestBody EmployeeDTO employeeDTO) {
        EmployeeDTO createdEmployee = employeeService.createEmployee(employeeDTO);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdEmployee);
    }
    
    @PutMapping("/{id}")
    @Operation(summary = "Update employee", description = "Update an existing employee record")
    @ApiResponse(responseCode = "200", description = "Employee successfully updated")
    @ApiResponse(responseCode = "400", description = "Invalid input data")
    @ApiResponse(responseCode = "404", description = "Employee not found")
    @ApiResponse(responseCode = "409", description = "Duplicate email address")
    public ResponseEntity<EmployeeDTO> updateEmployee(
            @Parameter(description = "Employee ID") @PathVariable Long id,
            @Parameter(description = "Updated employee data") @Valid @RequestBody EmployeeDTO employeeDTO) {
        EmployeeDTO updatedEmployee = employeeService.updateEmployee(id, employeeDTO);
        return ResponseEntity.ok(updatedEmployee);
    }
    
    @DeleteMapping("/{id}")
    @Operation(summary = "Delete employee", description = "Delete an employee record")
    @ApiResponse(responseCode = "204", description = "Employee successfully deleted")
    @ApiResponse(responseCode = "404", description = "Employee not found")
    @ApiResponse(responseCode = "400", description = "Cannot delete employee with active subordinates")
    public ResponseEntity<Void> deleteEmployee(
            @Parameter(description = "Employee ID") @PathVariable Long id) {
        employeeService.deleteEmployee(id);
        return ResponseEntity.noContent().build();
    }
    
    @PutMapping("/{id}/terminate")
    @Operation(summary = "Terminate employee", description = "Mark employee as terminated")
    @ApiResponse(responseCode = "200", description = "Employee successfully terminated")
    @ApiResponse(responseCode = "404", description = "Employee not found")
    public ResponseEntity<EmployeeDTO> terminateEmployee(
            @Parameter(description = "Employee ID") @PathVariable Long id) {
        EmployeeDTO terminatedEmployee = employeeService.terminateEmployee(id);
        return ResponseEntity.ok(terminatedEmployee);
    }
    
    @PutMapping("/{id}/reactivate")
    @Operation(summary = "Reactivate employee", description = "Reactivate a terminated employee")
    @ApiResponse(responseCode = "200", description = "Employee successfully reactivated")
    @ApiResponse(responseCode = "404", description = "Employee not found")
    public ResponseEntity<EmployeeDTO> reactivateEmployee(
            @Parameter(description = "Employee ID") @PathVariable Long id) {
        EmployeeDTO reactivatedEmployee = employeeService.reactivateEmployee(id);
        return ResponseEntity.ok(reactivatedEmployee);
    }
    
    @GetMapping("/departments")
    @Operation(summary = "Get all departments", description = "Retrieve list of all departments")
    @ApiResponse(responseCode = "200", description = "Successfully retrieved departments")
    public ResponseEntity<List<String>> getAllDepartments() {
        List<String> departments = employeeService.getAllDepartments();
        return ResponseEntity.ok(departments);
    }
    
    @GetMapping("/by-department/{department}")
    @Operation(summary = "Get employees by department", 
               description = "Retrieve all employees in a specific department")
    @ApiResponse(responseCode = "200", description = "Successfully retrieved employees")
    public ResponseEntity<List<EmployeeDTO>> getEmployeesByDepartment(
            @Parameter(description = "Department name") @PathVariable String department) {
        List<EmployeeDTO> employees = employeeService.getEmployeesByDepartment(department);
        return ResponseEntity.ok(employees);
    }
    
    @GetMapping("/{managerId}/subordinates")
    @Operation(summary = "Get subordinates", description = "Retrieve all subordinates of a manager")
    @ApiResponse(responseCode = "200", description = "Successfully retrieved subordinates")
    public ResponseEntity<List<EmployeeDTO>> getSubordinates(
            @Parameter(description = "Manager ID") @PathVariable Long managerId) {
        List<EmployeeDTO> subordinates = employeeService.getSubordinates(managerId);
        return ResponseEntity.ok(subordinates);
    }
    
    // Audit endpoints
    @GetMapping("/{id}/revisions")
    @Operation(summary = "Get employee audit history", 
               description = "Retrieve all audit revisions for an employee")
    @ApiResponse(responseCode = "200", description = "Successfully retrieved audit history")
    @ApiResponse(responseCode = "404", description = "Employee not found")
    public ResponseEntity<Revisions<Long, Employee>> getEmployeeRevisions(
            @Parameter(description = "Employee ID") @PathVariable Long id) {
        Revisions<Long, Employee> revisions = employeeService.getEmployeeRevisions(id);
        return ResponseEntity.ok(revisions);
    }
    
    @GetMapping("/{id}/revisions/{revisionNumber}")
    @Operation(summary = "Get specific employee revision", 
               description = "Retrieve a specific audit revision for an employee")
    @ApiResponse(responseCode = "200", description = "Successfully retrieved revision")
    @ApiResponse(responseCode = "404", description = "Employee or revision not found")
    public ResponseEntity<Revision<Long, Employee>> getEmployeeRevision(
            @Parameter(description = "Employee ID") @PathVariable Long id,
            @Parameter(description = "Revision number") @PathVariable Long revisionNumber) {
        Optional<Revision<Long, Employee>> revision = 
            employeeService.getEmployeeRevision(id, revisionNumber);
        return revision.map(ResponseEntity::ok)
                      .orElse(ResponseEntity.notFound().build());
    }
    
    @GetMapping("/{id}/revisions/last")
    @Operation(summary = "Get last employee revision", 
               description = "Retrieve the most recent audit revision for an employee")
    @ApiResponse(responseCode = "200", description = "Successfully retrieved last revision")
    @ApiResponse(responseCode = "404", description = "Employee not found")
    public ResponseEntity<Revision<Long, Employee>> getLastEmployeeRevision(
            @Parameter(description = "Employee ID") @PathVariable Long id) {
        Revision<Long, Employee> lastRevision = employeeService.getLastEmployeeRevision(id);
        return ResponseEntity.ok(lastRevision);
    }
}
```

## Deployment Configuration

### Kubernetes Deployment (deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: employee-service
  namespace: employee-ns
  labels:
    app: employee-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: employee-service
  template:
    metadata:
      labels:
        app: employee-service
        version: v1
    spec:
      containers:
      - name: employee-service
        image: company-registry.azurecr.io/employee-service:latest
        ports:
        - containerPort: 8080
          protocol: TCP
        env:
        - name: DB_HOST
          value: "sql-server-service"
        - name: DB_PORT
          value: "1433"
        - name: DB_NAME
          value: "employee_db"
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /api/v1/actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /api/v1/actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
      imagePullSecrets:
      - name: acr-credentials
```

### Kubernetes Service (service.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: employee-service
  namespace: employee-ns
  labels:
    app: employee-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: employee-service
```

### GitHub Actions Workflow (.github/workflows/ci-cd.yml)
```yaml
name: Employee Service CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: company-registry.azurecr.io
  IMAGE_NAME: employee-service
  AKS_CLUSTER_NAME: company-aks-cluster
  AKS_RESOURCE_GROUP: company-resources
  NAMESPACE: employee-ns

jobs:
  test:
    runs-on: ubuntu-latest
    name: Test Application
    
    services:
      sqlserver:
        image: mcr.microsoft.com/mssql/server:2019-latest
        env:
          ACCEPT_EULA: Y
          SA_PASSWORD: YourStrong@Passw0rd
        ports:
          - 1433:1433
        options: --health-cmd="/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P YourStrong@Passw0rd -Q 'SELECT 1'" --health-interval=10s --health-timeout=5s --health-retries=3
    
    steps:
    - name: Checkout Code
      uses: actions/checkout@v4
    
    - name: Set up JDK 21
      uses: actions/setup-java@v4
      with:
        distribution: 'temurin'
        java-version: '21'
        cache: 'maven'
    
    - name: Run Tests
      run: |
        mvn clean test
        mvn jacoco:report
      env:
        DB_HOST: localhost
        DB_PORT: 1433
        DB_NAME: tempdb
        DB_USERNAME: sa
        DB_PASSWORD: YourStrong@Passw0rd
    
    - name: Upload Coverage Reports
      uses: codecov/codecov-action@v3
      with:
        file: ./target/site/jacoco/jacoco.xml

  build-and-deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    name: Build and Deploy
    
    steps:
    - name: Checkout Code
      uses: actions/checkout@v4
    
    - name: Set up JDK 21
      uses: actions/setup-java@v4
      with:
        distribution: 'temurin'
        java-version: '21'
        cache: 'maven'
    
    - name: Build Application
      run: |
        mvn clean compile
        mvn package -DskipTests
    
    - name: Azure Container Registry Login
      uses: azure/docker-login@v1
      with:
        login-server: ${{ env.REGISTRY }}
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
    
    - name: Build and Push Docker Image
      run: |
        mvn compile jib:build \
          -Djib.to.image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
          -Djib.to.tags=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
    
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Get AKS Credentials
      run: |
        az aks get-credentials \
          --resource-group ${{ env.AKS_RESOURCE_GROUP }} \
          --name ${{ env.AKS_CLUSTER_NAME }}
    
    - name: Deploy to AKS
      run: |
        # Update image in deployment
        kubectl set image deployment/employee-service \
          employee-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
          -n ${{ env.NAMESPACE }}
        
        # Wait for rollout to complete
        kubectl rollout status deployment/employee-service -n ${{ env.NAMESPACE }}
        
        # Verify deployment
        kubectl get pods -n ${{ env.NAMESPACE }} -l app=employee-service

  security-scan:
    runs-on: ubuntu-latest
    name: Security Scan
    
    steps:
    - name: Checkout Code
      uses: actions/checkout@v4
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
```

## Key Configuration Changes Needed

### 1. SQL Server Configuration
```sql
-- Enable SQL Server for Spring Batch (if using sequences)
CREATE SEQUENCE BATCH_STEP_EXECUTION_SEQ START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE BATCH_JOB_EXECUTION_SEQ START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE BATCH_JOB_SEQ START WITH 1 INCREMENT BY 1;
```

### 2. AKS Namespace Setup
```bash
# Create namespace
kubectl create namespace employee-ns

# Create database credentials secret
kubectl create secret generic db-credentials \
  --from-literal=username=employee_user \
  --from-literal=password=SecurePassword123! \
  -n employee-ns

# Create ACR credentials secret
kubectl create secret docker-registry acr-credentials \
  --docker-server=company-registry.azurecr.io \
  --docker-username=<acr-username> \
  --docker-password=<acr-password> \
  -n employee-ns
```

### 3. Database Schema Initialization
```sql
-- Create database
CREATE DATABASE employee_db;
USE employee_db;

-- Create user for the application
CREATE LOGIN employee_user WITH PASSWORD = 'SecurePassword123!';
CREATE USER employee_user FOR LOGIN employee_user;

-- Grant necessary permissions
ALTER ROLE db_datareader ADD MEMBER employee_user;
ALTER ROLE db_datawriter ADD MEMBER employee_user;
ALTER ROLE db_ddladmin ADD MEMBER employee_user;

-- Run DDL scripts for employee table and audit tables
-- (Execute the DDL scripts provided above)
```

## Monitoring and Observability

### Application Metrics Configuration
```yaml
# Additional application.yml configuration for monitoring
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always
      show-components: always
    metrics:
      enabled: true
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
        step: 10s
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
      sla:
        http.server.requests: 10ms, 50ms, 100ms, 200ms, 500ms, 1s, 2s, 5s

# Custom application metrics
application:
  metrics:
    enabled: true
    custom:
      employee-operations: true
      audit-operations: true
```

## Testing Strategy

### Integration Test Example
```java
package com.company.microservices.employee.integration;

import com.company.microservices.employee.dto.EmployeeDTO;
import com.company.microservices.employee.entity.EmployeeStatus;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureWebMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.TestPropertySource;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;
import org.testcontainers.containers.MSSQLServerContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;
import java.time.LocalDate;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureWebMvc
@Testcontainers
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:tc:sqlserver:2019-latest:///testdb",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
@Transactional
public class EmployeeControllerIntegrationTest {
    
    @Container
    static MSSQLServerContainer<?> sqlServer = new MSSQLServerContainer<>("mcr.microsoft.com/mssql/server:2019-latest")
            .acceptLicense();
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    public void testCreateEmployee() throws Exception {
        EmployeeDTO employeeDTO = new EmployeeDTO();
        employeeDTO.setEmployeeNumber("TEST001");
        employeeDTO.setFirstName("John");
        employeeDTO.setLastName("Doe");
        employeeDTO.setEmail("john.doe@test.com");
        employeeDTO.setHireDate(LocalDate.now());
        employeeDTO.setJobTitle("Developer");
        employeeDTO.setDepartment("Engineering");
        employeeDTO.setSalary(new BigDecimal("75000.00"));
        
        mockMvc.perform(post("/api/v1/employees")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(employeeDTO)))
                .andExpect(status().isCreated())
                .andExpected(jsonPath("$.employeeNumber").value("TEST001"))
                .andExpected(jsonPath("$.firstName").value("John"))
                .andExpected(jsonPath("$.lastName").value("Doe"))
                .andExpected(jsonPath("$.status").value("ACTIVE"));
    }
    
    @Test
    public void testGetEmployeeById() throws Exception {
        // First create an employee
        EmployeeDTO employeeDTO = createTestEmployee();
        
        mockMvc.perform(get("/api/v1/employees/1"))
                .andExpect(status().isOk())
                .andExpected(jsonPath("$.employeeNumber").value("TEST001"));
    }
    
    private EmployeeDTO createTestEmployee() {
        EmployeeDTO employeeDTO = new EmployeeDTO();
        employeeDTO.setEmployeeNumber("TEST001");
        employeeDTO.setFirstName("John");
        employeeDTO.setLastName("Doe");
        employeeDTO.setEmail("john.doe@test.com");
        employeeDTO.setHireDate(LocalDate.now());
        employeeDTO.setStatus(EmployeeStatus.ACTIVE);
        return employeeDTO;
    }
}
```

## Summary

This comprehensive design document provides:

1. **Complete Spring Boot 3.2 application** with JDK 21 support
2. **Full CRUD REST API** for Employee management
3. **Spring Data Envers integration** for complete audit tracking
4. **SQL Server database** with proper DDL/DML scripts
5. **AKS deployment configuration** with Kubernetes manifests
6. **GitHub Actions CI/CD pipeline** for automated deployment
7. **Comprehensive testing strategy** with integration tests
8. **Production-ready monitoring** and observability

The solution follows microservices best practices and provides a solid foundation for migrating legacy mainframe functionality to modern cloud-native architecture.