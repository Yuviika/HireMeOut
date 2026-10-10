# Hire Me Out

A job portal we built with Java and Spring Boot. There are three kinds of users, job seekers, employers and an admin, and each of them gets their own dashboard.

The flow is simple. An employer posts a role, the admin approves it, seekers apply, and the employer moves each candidate through the stages (applied, under review, interview, offer or rejected). The seeker and the employer can message each other on an application. The dashboards also have a small live bar that shows what just happened (a new job, a new application, a status change) without refreshing the page.


## What's inside

I wanted one project that actually uses the core Java topics instead of a pile of small exercises, so here is where each one shows up:

- **Inheritance and interfaces:** `PortalUser` is an abstract class and `Seeker`, `Employer` and `Admin` extend it. They all implement a `DashboardProvider` interface, so each role has its own dashboard title and greeting.
- **Enums:** every status in `ApplicationStatus` has its own label and decides which statuses it can move to. That is why a candidate can't jump from "applied" straight to "offer".
- **Generics and collections:** `DashboardSnapshot<T>` holds a typed list. A seeker's dashboard holds a `List<ApplicationView>` and the admin's holds a `List<Job>`.
- **Exceptions:** I made my own exceptions (`NotFoundException`, `ValidationException`, `ForbiddenException`, `ConflictException`). They carry a message that is safe to show, so the user sees "You have already applied to this job" instead of a stack trace.
- **JDBC:** the repositories use Spring's `JdbcTemplate` with plain SQL and `?` placeholders. No JPA or Hibernate.
- **Web:** Spring MVC controllers, Thymeleaf pages, and form validation.
- **Security:** each role can only open its own pages, CSRF protection is on, and there is optional Google and GitHub login.
- **Live updates:** STOMP over WebSocket.

## Tech

Java 21, Spring Boot 3.3, Thymeleaf, Spring JDBC, Spring Security, WebSocket, MySQL 8, Maven, JUnit 5.

## Running it

You need JDK 21 and MySQL 8. Maven is also needed, but IntelliJ has its own, so you can skip installing it if you use IntelliJ.

1. Start MySQL. You don't have to create the database, the app creates `hire_me_out` the first time it runs.
2. Copy `src/main/resources/application-example.properties` to `src/main/resources/application.properties` and put your MySQL password in. This file is in `.gitignore`, so it won't get committed.
3. Run `HireMeOutApplication` from your IDE, or use `mvn spring-boot:run`.
4. Open http://localhost:8080

If you'd rather not keep your password in a file, set `spring.datasource.password=${DB_PASSWORD:}` and define a `DB_PASSWORD` environment variable instead.

Every time it starts, the app runs `schema.sql` (tables are only created if they're missing) and `data.sql` (the demo rows use `INSERT IGNORE`, so restarting doesn't duplicate anything).

## Demo accounts

There are no passwords yet. The home page has three cards, and clicking one signs you in as a ready-made demo user.

- **Seeker (Aarav Sharma):** browse approved jobs, apply once per job, track each application, message the employer.
- **Employer (Maya Kapoor, Studio North):** post roles, see all her postings and whether they're approved, move candidates forward, message them.
- **Admin (Riya Sen):** approve or reject jobs that are waiting for review.

The seed data has one approved job ("Junior Java Developer"), one job waiting for approval ("Product Design Intern") and one application from Aarav that is already under review.

A new role posted by an employer stays hidden from seekers until the admin approves it. If someone who isn't signed in opens a protected page, they get sent back to the home page. If you're signed in as one role and try to open another role's dashboard, you get a 403 page.

## Google and GitHub login (optional)

The demo cards work without this. If you want real login:

1. Create OAuth credentials with Google and/or GitHub and add these redirect URIs:
   - `http://localhost:8080/login/oauth2/code/google`
   - `http://localhost:8080/login/oauth2/code/github`
2. In your `application.properties`, uncomment the `client-id` and `client-secret` lines for the provider from the example file and fill in the real values. Don't leave them blank, an empty `client-id` can stop the app from starting.
3. Restart. A button for each provider you set up shows in the nav bar.

When someone signs in, the app finds the user by email, or creates a new seeker if there isn't one. If the provider doesn't share an email (GitHub can hide it), they see an error page instead.

## Project layout

```
src/main/java/com/hiremeout
  config/            security, WebSocket, the signed-in user
  controller/        routes and forms
  domain/            Job, Application, Message, roles, statuses, DashboardSnapshot<T>
  domain/exception/  the custom exceptions
  repository/        JDBC queries
  service/           the business rules and live events
  PortalController   my first version of the controller, switched off and no longer used
src/main/resources
  templates/         Thymeleaf pages (fragments.html has the shared nav and footer)
  static/            css/app.css and js/live.js
  schema.sql, data.sql, application-example.properties
src/test/java        tests
```

## Tests

`mvn clean verify` runs them, and none of them need MySQL.

- `ApplicationStatusTest` and `PortalUserTest` cover the status rules, the user factory and the exceptions.
- `HiringJourneyServiceTest` goes through the whole flow (post, approve, apply, move through the stages, message) and also checks the rules that should block things, like applying twice. It uses small in-memory fake repositories instead of a real database.
- `SecurityConfigTest` checks the access rules and CSRF with MockMvc.

The controllers, pages and SQL aren't covered by automated tests. I check those by running the app and clicking through it.

## If it doesn't start

- `Access denied for user 'root'`: wrong username or password in `application.properties`.
- `Communications link failure`: MySQL isn't running, or it's on a different port than 3306.
- `release version 21 not supported`: the project SDK is older than 21. In IntelliJ go to File, Project Structure, Project and pick JDK 21.
- `Port 8080 was already in use`: close whatever is using it, or add `server.port=8081` to `application.properties`.
- `'mvn' is not recognized`: Maven isn't on your PATH. Use IntelliJ's Maven panel, or install Maven.
- Page looks plain and unstyled: press F12, open the Network tab, reload, and check the status of `app.css`.

## Not done yet

- The demo login isn't real authentication, anyone can become any role with a click. Before using this for real I'd add registration with passwords and remove `data.sql`.
- No resume upload. Applications are saved with a cover note only.
- The live bar goes to every open browser. It only shows job titles and stage names, never personal details.
- The admin can't read message threads, only the seeker and employer on that application can.
- Job lists aren't paginated.

Please don't commit `application.properties` or any OAuth secret.
