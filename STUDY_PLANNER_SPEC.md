# Study Planner with AI: Project Spec

A web app to manage courses, tasks, and study time, and to turn lecture notes into quizzes, summaries, and answers (RAG).

## Goals
- One real, deployed project where every new skill gets applied
- Portfolio quality: tests, CI/CD, Docker, live demo, clear README
- AI used as a feature with validation, not as a gimmick

## Tech Stack
| Area | Choice |
|---|---|
| Backend | Java 21, Spring Boot 3, Spring Web, Spring Data JPA, Spring Security (JWT), Bean Validation |
| Database | PostgreSQL + Flyway migrations, pgvector extension (for RAG later) |
| Testing | JUnit 5, Mockito, Testcontainers, MockMvc |
| DevOps | Docker, docker-compose, GitHub Actions |
| Frontend | React + TypeScript (or Angular), kept simple |
| AI | Claude API (called from a Spring service behind an interface) |
| Docs | OpenAPI/Swagger, README with architecture diagram |

## Data Model

**User**: id, email (unique), passwordHash, createdAt

**Course**: id, userId, name, semester, ects, examDate, color

**Task**: id, courseId, title, description, dueDate, priority (LOW/MEDIUM/HIGH), done, createdAt

**StudySession**: id, courseId, minutes, date, note

**Note**: id, courseId, title, content, createdAt, updatedAt

**NoteChunk** (for RAG): id, noteId, chunkIndex, text, embedding (vector)

**Quiz**: id, noteId, createdAt

**Question**: id, quizId, text, optionA, optionB, optionC, optionD, correctOption, explanation

**QuizAttempt**: id, quizId, score, takenAt

Rules: every row is reachable only through the owning user. Always check ownership in the service layer, not only in the controller.

## API Endpoints

### Auth
- `POST /api/auth/register`
- `POST /api/auth/login` returns JWT

### Courses
- `GET /api/courses`, `POST /api/courses`
- `GET/PUT/DELETE /api/courses/{id}`

### Tasks
- `GET /api/courses/{id}/tasks`, `POST /api/courses/{id}/tasks`
- `PUT/DELETE /api/tasks/{id}`
- `PATCH /api/tasks/{id}/done`
- `GET /api/tasks/upcoming?days=7`

### Study sessions and stats
- `POST /api/sessions`
- `GET /api/stats/weekly` (minutes per course, per day)
- `GET /api/stats/courses/{id}` (progress, tasks done, total time)

### Notes
- `GET/POST /api/courses/{id}/notes`
- `PUT/DELETE /api/notes/{id}`

### AI features
- `POST /api/notes/{id}/summary`
- `POST /api/notes/{id}/quiz` generates and stores a quiz
- `POST /api/quizzes/{id}/attempts` submits answers, returns score
- `POST /api/ask` body: `{ "question": "...", "courseId": 1 }` returns answer plus source notes
- `POST /api/courses/{id}/study-plan` suggests a weekly schedule from exam dates and open tasks

### Error format (use everywhere)
```json
{ "timestamp": "...", "status": 400, "error": "Validation failed", "details": ["title must not be blank"] }
```

## Project Structure (layered)
```
study-planner/
  backend/
    src/main/java/.../studyplanner/
      auth/          (controller, service, JwtService, security config)
      course/        (entity, repository, service, controller, dto)
      task/
      session/
      note/
      quiz/
      ai/            (AiClient interface, ClaudeAiClient, prompts, response parsing)
      rag/           (chunking, embeddings, retrieval)
      common/        (exceptions, error handler, config)
    src/main/resources/db/migration/   (Flyway V1__init.sql ...)
    src/test/java/...
    Dockerfile
  frontend/
  docker-compose.yml
  .github/workflows/ci.yml
  README.md
```

## AI Design Rules
- Hide the model behind an `AiClient` interface so it can be mocked in tests and swapped later
- Ask for JSON output, parse it into DTOs, and validate (e.g. each question has exactly 4 options and a valid correct answer). If invalid, retry once, then return a clear error
- Never trust AI output: limit input length, handle timeouts, log failures
- RAG answers must show their source notes; if nothing relevant is found, say so instead of guessing
- Keep the API key in environment variables, never in the repo
- Add a per-user rate limit on AI endpoints to control cost

## 12-Week Plan

| Week | Focus | Deliverable |
|---|---|---|
| 1 | Setup: repo, Spring Boot, PostgreSQL via docker-compose, Flyway, first entity | Course CRUD working locally |
| 2 | Tasks and sessions, validation, global error handler, DTOs | All core CRUD + OpenAPI docs |
| 3 | Unit and integration tests (JUnit, Mockito, Testcontainers) | 70%+ coverage on services |
| 4 | Spring Security + JWT, ownership checks | Login works, users are isolated |
| 5 | Dockerfile for the app, GitHub Actions (build + test) | Green CI badge in README |
| 6 | Stats endpoints (weekly, per course), upcoming tasks | Meaningful SQL/JPQL queries |
| 7 | Frontend v1: login, courses, tasks, weekly view | Usable UI |
| 8 | Deploy (Render, Azure, or a VPS) + env config | Live demo link |
| 9 | AI 1: note summary + quiz generation with validation | Quiz flow end to end |
| 10 | AI 2: chunking, embeddings, pgvector | Retrieval working in tests |
| 11 | AI 3: "ask my notes" with sources, study plan suggestion | RAG endpoint + UI |
| 12 | Polish: README, architecture diagram, screenshots, cleanup, demo video | Portfolio-ready |

Each week: finish one deliverable, commit in small steps, and write 3 sentences in a `LEARNINGS.md` about what you learned and what was hard. You can use those in interviews.

## Definition of Done (v1)
- [ ] App runs with one `docker-compose up`
- [ ] CI builds and tests on every push
- [ ] Authentication and per-user data isolation tested
- [ ] At least 3 AI features working with validation and error handling
- [ ] Deployed, with a live link
- [ ] README: problem, features, stack, architecture diagram, how to run, screenshots, "what I would improve"

## Stretch Ideas (after v1)
- Spaced repetition for quiz questions
- Calendar export (.ics) for exams and deadlines
- Streaming AI responses
- Caching of AI results
- Basic monitoring (Spring Actuator + logs)
