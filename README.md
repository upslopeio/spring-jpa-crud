# Spring CRUD

## Overview: REST

In order to create a CRUD API you typically need at least 5 endpoints:

1. List all items
1. Show a single item
1. Create an item
1. Update an item
1. Delete an item

The URLs for these typically look like this:

| Action | Verb           | Path                  |
|--------|----------------|-----------------------|
| list   | GET            | `/backlog-items`      |
| show   | GET            | `/backlog-items/{id}` |
| create | POST           | `/backlog-items`      |
| update | PUT (or PATCH) | `/backlog-items/{id}` |
| delete | DELETE         | `/backlog-items/{id}` |

## Overview: Spring / JPA Setup

In order to create a project for the first time you need to:

1. Generate a Spring Application with Web, JPA and Postgres dependencies
1. Create a database 
1. Add the database connection configurations

NOTE: You shouldn't check in database names, username and passwords. So in this repo, the properties are split into two files:

`src/main/resources/application.properties`

```properties
spring.profiles.active=default
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=none
spring.jpa.hibernate.show-sql=true
```

`src/main/resources/application-default.properties`

```properties
spring.datasource.url=jdbc:postgresql://localhost/spring_crud_backlog
spring.datasource.username=
spring.datasource.password=
```

## Overview: Spring / JPA CRUD endpoints

In order to create a CRUD endpoint you need to:

1. Create a database table
1. Create an `@Entity` class with the same structure as the table
1. Create a `CrudRepository` interface
1. Create a `@RestController` that injects the `CrudRepository`
1. Add 5 methods, using `@GetMapping`, `@PostMapping`, `@PutMapping` and `@DeleteMapping`

---
---
---

# Step-by-Step Instructions

## Create a database

From a Terminal, run the following command to create a database:

```
createdb spring_crud_backlog
```

NOTE: you may need to specify the user, like this `createdb -U postgres spring_crud_backlog`

## Specify your connection settings

1. Copy `application-default.properties.tmpl` to `application-default.properties`
   ```
   cp src/main/resources/application-default.properties.tmpl src/main/resources/application-default.properties
   ```
1. Update your username / password in `src/main/resources/application-default.properties`

## Run the app to make sure it works

```
./gradlew bootRun
```

Then open http://localhost:8080.

You should see "Whitelabel Error Page", and you should not see any error messages in your console.

## Create a table

1. Create a file named `sql/001-create-stories.sql`
1. Write the sql to create the table:
     ```sql
     create table backlog_items (
         id uuid primary key,
         title varchar not null,
         type varchar not null,
         status varchar not null
     );    
     ```
1. Execute the SQL file against your local database:
   ```
   psql -f sql/001-create-stories.sql -d spring_crud_backlog
   ```

## Create an @Entity class

```java
package com.example.demo;

import javax.persistence.*;
import java.util.*;

@Entity
@Table(name = "backlog_items")
public class BacklogItem {
    // this tells hibernate to generate new UUIDs for all new backlog items
    @Id
    @GeneratedValue
    private UUID id;

    // for each column, you add a private field, getter and setter
    @Column
    private String title;

    @Column
    private String type;

    @Column
    private String status;

    public UUID getId() {
        return id;
    }

    public void setId(UUID id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}
```

1. Re-run `./gradlew bootRun` and make sure there are no compilation errors
1. Visit http://localhost:8080
1. You should still see a "Whitelabel Error Page"

## Create a CrudRepository Interface

```java
package com.example.demo;

import org.springframework.data.repository.CrudRepository;

import java.util.*;

// BacklogItem is the name of the @Entity class defined above
// UUID matches the type of the primary key field from the @Entity class
public interface BacklogItemRepository extends CrudRepository<BacklogItem, UUID> {

}
```

## Create the Controller with the list method

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

// tells Spring that this class is a controller class and can handle HTTP requests
@RestController
// the "base path" for all of the paths defined below
@RequestMapping("/backlog-items")
public class BacklogItemController {

   // a field that can hold a reference to the CrudRepository
   private final BacklogItemRepository db;

   // A constructor that tells Spring to inject this CrudRepository into the controller
   public BacklogItemController(BacklogItemRepository db) {
      this.db = db;
   }

   // @GetMapping("") in combination with @RequestMapping above 
   // means that the URL that will target this method is /backlog-items
   @GetMapping("")
   public Iterable<BacklogItem> list() {
      return this.db.findAll();
   }
}
```

1. Stop / restart `./gradlew bootRun` and open the browser
1. Visit http://localhost:8080/backlog-items
1. You should see `[]` which is an empty JSON array (because the table has no rows)

## Create the insert method

Add the following method to your controller:

```java
@PostMapping("")
public BacklogItem create(@RequestBody BacklogItem item) {
        return this.db.save(item);
}
```

1. Stop & restart `./gradlew bootRun`
1. Execute the following curl command:
   ```
   curl -X POST -H "content-type: application/json" -d '{"title": "first item", "type": "story", "status": "unstarted"}' http://localhost:8080/backlog-items
   ```
1. Visit http://localhost:8080/backlog-items
1. You should see your backlog item in JSON format

## Create the show method

Add the following method to the controller:

```java
@GetMapping("{id}")
public BacklogItem show(@PathVariable UUID id) {
     return this.db.findById(id).orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "not found"));
}
```

1. Stop & restart `./gradlew bootRun`
1. Visit http://localhost:8080/backlog-items/<some id>
1. You should see your backlog item in JSON format
1. Visit http://localhost:8080/backlog-items/foobarbaz (an ID that doesn't exist)
1. You should see a 404 error in a "Whitelabel Error Page"

## Create the PUT method

Add the following method to your controller:

```java
@PutMapping("{id}")
public BacklogItem update(@PathVariable UUID id, @RequestBody BacklogItem item) {
        item.setId(id);
        return this.db.save(item);
}
```

1. Stop & restart `./gradlew bootRun`
1. Execute the following curl command and replace `<SOME ID>` with an actual ID of a backlog item:
   ```
   curl -X PUT -H "content-type: application/json" -d '{"title": "first item", "type": "story", "status": "started"}' http://localhost:8080/backlog-items/<SOME ID>
   ```
1. Visit http://localhost:8080/backlog-items
1. You should see that the backlog item is `"started"`

## Create the delete method

Add the following method to your controller:

```java
@DeleteMapping("{id}")
public ResponseEntity<?> delete(@PathVariable UUID id){
        this.db.deleteById(id);
        return ResponseEntity.ok().build();
}
```

1. Stop & restart `./gradlew bootRun`
1. Execute the following curl command and replace `<SOME ID>` with an actual ID of a backlog item:
   ```
   curl -X DELETE -H "content-type: application/json" http://localhost:8080/backlog-items/<SOME ID>
   ```
1. Visit http://localhost:8080/backlog-items
1. You should see that the backlog item is gone


### Reference Documentation

For further reference, please consider the following sections:

* [Official Gradle documentation](https://docs.gradle.org)
* [Spring Boot Gradle Plugin Reference Guide](https://docs.spring.io/spring-boot/docs/2.4.5/gradle-plugin/reference/html/)
* [Create an OCI image](https://docs.spring.io/spring-boot/docs/2.4.5/gradle-plugin/reference/html/#build-image)
* [Spring Boot DevTools](https://docs.spring.io/spring-boot/docs/2.4.5/reference/htmlsingle/#using-boot-devtools)
* [Spring Web](https://docs.spring.io/spring-boot/docs/2.4.5/reference/htmlsingle/#boot-features-developing-web-applications)
* [Spring Data JPA](https://docs.spring.io/spring-boot/docs/2.4.5/reference/htmlsingle/#boot-features-jpa-and-spring-data)

### Guides

The following guides illustrate how to use some features concretely:

* [Building a RESTful Web Service](https://spring.io/guides/gs/rest-service/)
* [Serving Web Content with Spring MVC](https://spring.io/guides/gs/serving-web-content/)
* [Building REST services with Spring](https://spring.io/guides/tutorials/bookmarks/)
* [Accessing Data with JPA](https://spring.io/guides/gs/accessing-data-jpa/)

### Additional Links

These additional references should also help you:

* [Gradle Build Scans – insights for your project's build](https://scans.gradle.com#gradle)

