# HIRE me out

A beginner-friendly Java web project: a role-based job portal with three kinds of user
(**seekers**, **employers** and **admins**) in one Spring Boot application backed by MySQL.

## What it demonstrates

| Topic | Where to look |
|---|---|
| **OOP: inheritance, abstract class, interface** | `PortalUser` is an abstract base class that implements the `DashboardProvider` interface. `Seeker`, `Employer` and `Admin` extend it and give each role its own dashboard title and greeting. `PortalUser.fromRow(...)` builds the right subclass for a database row. |
| **Polymorphism** | Every constant of the `ApplicationStatus` enum overrides `label()` and `canMoveTo(...)`, so each stage has its own text and its own rules for what can come next. |
| **Collections and generics** | `DashboardSnapshot<T>` holds a typed `List<T>`: a seeker's snapshot holds `List<ApplicationView>`, an admin's holds `List<Job>`. Records such as `Job`, `Application` and `Message` replace untyped maps. |
| **Exceptions** | `PortalException` and its subclasses (`NotFoundException` 404, `ValidationException` 400, `ForbiddenException` 403, `ConflictException` 409) carry a friendly message. Controllers show those messages on the page, and `GlobalExceptionHandler` turns anything unexpected into a generic error page. |
| **JDBC** | The four classes in `repository/` use Spring's `JdbcTemplate` with parameterized SQL only (no string-built queries, no JPA). |
| **Web layer** | Spring MVC controllers render Thymeleaf pages. Forms are validated with Bean Validation, and a successful form submission ends in a redirect so refreshing the page never re-submits it. |
| **Security** | Role-based access rules, CSRF protection, a demo login, and optional Google/GitHub sign-in. See `config/SecurityConfig`. |
| **Live updates** | After a job is posted or reviewed, a candidate applies, or an application changes stage, `LiveEventPublisher` sends a short message over STOMP/WebSocket to `/topic/updates`, and `static/js/live.js` shows it on open dashboards. |

## Requirements

- JDK 21
- Maven 3.9+ (IntelliJ IDEA bundles its own Maven, so you do not need to install it separately)
- MySQL 8+

## Run locally

1. Make sure MySQL is running and you know your password. You do not need to create the
   database: the app creates `hire_me_out` on first start.
2. Copy `src/main/resources/application-example.properties` to
   `src/main/resources/application.properties` and put your MySQL password in it.
   That file is listed in `.gitignore`, so it will not be committed.
   (Prefer not to write the password in a file? Use `spring.datasource.password=${DB_PASSWORD:}`
   and set a `DB_PASSWORD` environment variable, for example in your IDE's run configuration.)
3. Start the app: run `HireMeOutApplication` from your IDE, or `mvn spring-boot:run`.
4. Open `http://localhost:8080`.

On every start the app runs `schema.sql` (tables are only created if missing) and `data.sql`
(demo rows use `INSERT IGNORE`, so restarting never duplicates them). This is controlled by
`spring.sql.init.mode=always`. Remove `data.sql` before a production launch.

## Demo roles

The home page has three cards. Each one signs you in as a ready-made demo user:

| Card | Demo user | What that role can do |
|---|---|---|
| Seeker | Aarav Sharma | Browse approved jobs, apply once per job, follow the status of each application, message the employer. |
| Employer | Maya Kapoor (Studio North) | Post roles (they wait for admin approval), see all postings and their status, move candidates through the stages, message candidates. |
| Admin | Riya Sen | Approve or reject pending jobs. |

The seed data also contains one approved job ("Junior Java Developer"), one pending job
("Product Design Intern") and one application from Aarav that is already under review.

## Pages and who can open them

| URL | Who | Purpose |
|---|---|---|
| `GET /` | everyone | Landing page with the demo role cards |
| `GET /jobs` | everyone | Public board of approved jobs, with search (`?q=`) |
| `POST /demo-login/{role}` | everyone | Demo sign-in (`seeker`, `employer`, `admin`) |
| `POST /logout` | signed in | Sign out |
| `GET /dashboard/seeker` | seekers | Own applications and open roles |
| `POST /jobs/{id}/apply` | seekers | Apply to an approved job |
| `GET /dashboard/employer` | employers | Postings, candidate queue, post-a-role form |
| `POST /jobs` | employers | Post a new role (starts as pending) |
| `POST /applications/{id}/status` | employers | Move a candidate to the next stage |
| `GET /dashboard/admin` | admins | Review queue |
| `POST /jobs/{id}/review` | admins | Approve or reject a pending job |
| `GET`/`POST /applications/{id}/messages` | the seeker and the employer of that application | Message thread |
| `/live` | browsers | WebSocket endpoint for live updates |

Anyone who is not signed in and opens a protected page is sent back to the home page.
A signed-in user who opens a page for another role gets a 403 page. Old `/demo/{role}`
links redirect to the matching dashboard.

## Optional: Google and GitHub sign-in

The demo login works without any of this. To turn on real sign-in:

1. Create OAuth credentials with Google and/or GitHub and register these redirect URIs:
   `http://localhost:8080/login/oauth2/code/google` and
   `http://localhost:8080/login/oauth2/code/github`.
2. In your `application.properties`, uncomment the matching `client-id` and `client-secret`
   lines from the example file and fill in real values. Do not leave them blank: an empty
   `client-id` can stop the app from starting.
3. Restart. A button for each provider you configured appears in the navigation bar.

When someone signs in, the app looks up the portal user by email, or creates a new **seeker**
if there is none. If the provider does not share an email address (GitHub can hide it),
the person sees a friendly error page instead.

## Project structure

```
src/main/java/com/hiremeout
  HireMeOutApplication.java   starts the app
  PortalController.java       OLD prototype controller, deprecated and switched off (see below)
  config/                     SecurityConfig, WebSocketConfig, AppUser, CurrentUser
  controller/                 web routes, forms (ApplyForm, PostJobForm, SendMessageForm),
                              GlobalExceptionHandler, LayoutModelAdvice
  domain/                     Job, Application, Message, roles, statuses, DashboardSnapshot<T>
  domain/exception/           PortalException and its four subclasses
  repository/                 JDBC database access (Job, Application, User, Message)
  service/                    business rules and live events
src/main/resources
  templates/                  Thymeleaf pages; fragments.html holds the shared nav, head and footer;
                              dashboard/ holds one partial per role
  static/                     css/app.css and js/live.js
  schema.sql                  MySQL tables
  data.sql                    demo seed data
  application-example.properties   copy to application.properties
src/test/java/com/hiremeout   tests (see below)
```

`PortalController.java` is the first prototype's controller. Its routes now live in the
`controller/` package, and it is no longer registered with Spring (two controllers cannot
map the same URL). It is kept only for reference and is safe to delete.

## Tests

Run them with `mvn clean verify`. **No MySQL is needed** for any of them.

- `domain/ApplicationStatusTest`, `domain/PortalUserTest`: status rules, parsing, the user
  factory, exception codes and the generic snapshot.
- `service/HiringJourneyServiceTest`: the whole journey (post, approve, apply, move through
  the stages, message) plus the main rule violations, using small in-memory fake repositories.
- `config/SecurityConfigTest`: access rules and CSRF, using MockMvc.

The controllers, templates and the SQL in the repositories are checked by running the app
and clicking through it, not by automated tests.

## Troubleshooting

| Message | Likely cause |
|---|---|
| `Access denied for user 'root'` | Wrong username or password in `application.properties`. |
| `Communications link failure` / `Connection refused` | MySQL is not running, or it is on a port other than 3306. |
| `release version 21 not supported` | The project SDK is older than JDK 21. In IntelliJ: File, Project Structure, Project, set the SDK to 21. |
| `Port 8080 was already in use` | Stop the other program, or add `server.port=8081` to `application.properties`. |
| `'mvn' is not recognized` | Maven is not on your PATH. Use IntelliJ's Maven panel instead, or install Maven. |
| Pages look plain / unstyled | In the browser press F12, open Network, reload, and check the status of `app.css`. |

## Known limitations

- **Demo login is not real authentication.** Anyone can become any role with one click.
  Before a real launch, replace it with registration and a credential flow (the OAuth sign-in
  is a start) and remove `data.sql`.
- Resume upload is not built; applications are saved without a resume file.
- Live updates go to every connected browser. They contain only job titles and stage names,
  never personal details.
- Admins cannot read message threads; only the seeker and employer of an application can.
- Lists are not paginated.

## Safety note

Never commit `application.properties` or any OAuth secret. Use environment variables or a
secrets manager when you deploy.
