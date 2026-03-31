# How to Use Spring Data MongoDB Projections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Spring, Java, Projection, Spring Data

Description: Learn how to use Spring Data MongoDB projections to return partial documents, reducing data transfer and improving query performance in Java applications.

---

Spring Data MongoDB projections allow you to retrieve only specific fields from MongoDB documents rather than fetching complete objects. This reduces network transfer, lowers memory usage, and often eliminates the need for separate DTO mapping code.

## Interface-Based Projections

The simplest projection type uses a Java interface with getters matching the fields you want:

```java
// Full document class
@Document(collection = "employees")
public class Employee {
    @Id
    private String id;
    private String firstName;
    private String lastName;
    private String email;
    private double salary;
    private String department;
}

// Projection interface - only firstName, lastName, email
public interface EmployeeSummary {
    String getFirstName();
    String getLastName();
    String getEmail();
}
```

Add the projection return type to the repository:

```java
public interface EmployeeRepository extends MongoRepository<Employee, String> {
    List<EmployeeSummary> findByDepartment(String department);
}
```

Spring Data automatically maps the result to the projection interface at query time.

## Class-Based (DTO) Projections

You can also use a concrete class with a constructor:

```java
public class EmployeeNameDTO {
    private final String firstName;
    private final String lastName;

    public EmployeeNameDTO(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }

    // getters
}
```

```java
public interface EmployeeRepository extends MongoRepository<Employee, String> {
    List<EmployeeNameDTO> findByDepartment(String department);
}
```

## Dynamic Projections

Use a generic return type to select the projection at call time:

```java
public interface EmployeeRepository extends MongoRepository<Employee, String> {
    <T> List<T> findByDepartment(String department, Class<T> type);
}

// In service
List<EmployeeSummary> summaries = repo.findByDepartment("Engineering", EmployeeSummary.class);
List<Employee> full = repo.findByDepartment("Engineering", Employee.class);
```

## Field-Level Projections with @Query

For fine-grained control, use the `fields` parameter in `@Query`:

```java
import org.springframework.data.mongodb.repository.Query;

public interface EmployeeRepository extends MongoRepository<Employee, String> {

    // Include only firstName, lastName (MongoDB projection)
    @Query(value = "{ 'department': ?0 }", fields = "{ 'firstName': 1, 'lastName': 1 }")
    List<Employee> findNamesByDepartment(String department);

    // Exclude salary field
    @Query(value = "{ 'department': ?0 }", fields = "{ 'salary': 0 }")
    List<Employee> findWithoutSalaryByDepartment(String department);
}
```

## Nested Field Projections

Interface projections support nested documents via nested interfaces:

```java
public interface EmployeeWithAddress {
    String getFirstName();
    String getLastName();
    AddressSummary getAddress();

    interface AddressSummary {
        String getCity();
        String getState();
    }
}
```

## Open Projections with @Value

Use `@Value` in projection interfaces to compute derived values:

```java
public interface EmployeeSummary {
    @org.springframework.beans.factory.annotation.Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
    String getEmail();
}
```

Note that open projections using `@Value` always load the full document since Spring EL is evaluated in memory, not pushed to MongoDB.

## Performance Considerations

Closed projections (where all getter methods match document fields directly) are pushed to MongoDB as field projections, reducing the data returned from the database. Open projections with `@Value` or default interface methods load the full document first.

```java
// Closed projection - efficient, pushes to MongoDB
public interface EmployeeSummary {
    String getFirstName();  // matches field name
    String getEmail();      // matches field name
}
```

## Summary

Spring Data MongoDB projections provide a flexible way to return partial document data. Use closed interface projections for performance-critical queries where only a subset of fields is needed, DTO projections for clean data transfer objects, and `@Query` with `fields` for cases where existing projection interfaces are insufficient. Dynamic projections offer the most flexibility when the required projection type varies at runtime.
