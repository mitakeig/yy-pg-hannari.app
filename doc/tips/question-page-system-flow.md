# Question page: frontend-to-backend investigation

Investigated: 2026-11-09. Repository revision: `c32b441`.

Scope: the quiz/question screen at `/contents/:id/quiz`, its content-page entry point, answer persistence, completion tracking, result page, and deployment routing. There is no separately named Question page in the application; the implementation is `QuizView.vue`.

Evidence: source inspection and a read-only check of content JSON. `docker compose ps` in `sys` returned no running services for this Compose project. No application requests, database queries, or E2E tests were executed. Behavior below is derived from code, not a claim about a running deployment. Application code was not changed.

## 1. End-to-end sequence

```mermaid
sequenceDiagram
    actor Learner
    participant Detail as ContentDetailView
    participant View as QuizView
    participant Store as Pinia learning store
    participant API as Axios API client
    participant Controller as NestJS controller / guards
    participant Service as QuizzesService
    participant DB as PostgreSQL via Prisma
    participant Result as QuizResultView

    Learner->>Detail: Open /contents/:id
    Detail->>API: GET /contents/:id
    API->>Controller: Read content (public endpoint)
    Controller->>DB: Content + category + quizzes + choices
    DB-->>Detail: Content response via service / API / store
    Detail->>API: POST /progress/:id (no body)
    API->>Controller: Bearer JWT
    Controller->>DB: Upsert progress with completed=false
    Learner->>Detail: Click クイズをはじめる
    Detail->>View: Vue Router /contents/:id/quiz
    View->>Store: resetQuizSession(); fetchQuizzes(id)
    Store->>API: GET /quizzes/content/:id
    API->>Controller: Bearer header if available (endpoint public)
    Controller->>Service: findByContentId(id)
    Service->>DB: Verify Content; fetch ordered quizzes / choices
    DB-->>Store: Quiz array via service / API
    Store-->>View: Render first question and choices
    Learner->>View: Click choice
    View->>Store: answerQuiz(quizId, choiceId)
    Store->>API: POST /quizzes/:quizId/answer {choiceId}
    API->>Controller: Bearer JWT
    Controller->>DB: JWT strategy looks up User by token sub
    Controller->>Service: submitAnswer(quizId, user.id, choiceId)
    Service->>DB: Read Quiz + choices; insert UserQuizAttempt
    Service->>DB: Count content quizzes; read distinct historically answered quiz IDs
    opt Every quiz has at least one historical answer
        Service->>DB: Upsert UserProgress completed=true
    end
    Service-->>Store: isCorrect, correctChoiceId, explanation, attemptId
    Store-->>View: Append local result; reveal feedback; lock choices
    Learner->>View: Click つぎへ
    View->>Store: Advance local index (no request)
    Learner->>View: After final answer click けっかを見る
    View->>Result: Navigate /quiz-result (no request)
    Result->>Store: Read quizResults; calculate current-session score
```

## 2. Frontend entry and authentication

| Step | Exact implementation | Source |
|---|---|---|
| App startup | Creates Vue app, installs Pinia and Vue Router, mounts `App`; `App` renders an unkeyed `RouterView`. | `sys/frontend/src/main.ts:7`, `sys/frontend/src/App.vue:6` |
| Content entry | On mount, awaits `fetchContent(contentId)`, then `markProgress(contentId)`. The second call still runs if the first fails because the store catches errors. | `sys/frontend/src/views/ContentDetailView.vue:25` |
| Start button | Rendered when `currentContent.quizzes.length > 0`; pushes `/contents/${contentId}/quiz`. No server-side session is created. | `sys/frontend/src/views/ContentDetailView.vue:84`, `:210` |
| Quiz route | Lazy-loads `QuizView.vue`; `meta.requiresAuth=true`. Result route is `/quiz-result`. | `sys/frontend/src/router/index.ts:31` |
| Route guard | Authentication means only that Pinia has a truthy token. Missing token redirects to `/login`; the originally requested quiz URL is not preserved. | `sys/frontend/src/router/index.ts:46`, `sys/frontend/src/stores/auth.ts:8` |
| Token transport | Axios reads `localStorage.token` on every request and adds `Authorization: Bearer <token>`. | `sys/frontend/src/api/index.ts:8` |
| Actual API authentication | Only answer POST is JWT-guarded in the quizzes controller. Reading quizzes and content is public. JWT extraction verifies expiration/signature and looks up the User by `payload.sub`; the authenticated DB user is put in `req.user`. | `sys/backend/src/quizzes/quizzes.controller.ts:26`, `sys/backend/src/auth/jwt.strategy.ts:15`, `sys/backend/src/contents/contents.controller.ts:4` |

The quiz screen itself does not fetch the user profile. Dashboard mount does. A stale/expired token can therefore pass the frontend route guard and load public questions, then fail on answer submission. Axios has no response interceptor to redirect or clear tokens on 401. `auth.fetchProfile()` clears a token on 401, but it is not called by the quiz page.

## 3. Initial question load

1. `QuizView` captures `route.params.id` once as `contentId` (`QuizView.vue:12`).
2. On mount it clears quizzes, results, and index with `resetQuizSession()`, then awaits `fetchQuizzes(contentId)` (`:17`; `learning.ts:90`).
3. `fetchQuizzes` sets shared `loading=true`, clears shared `error`, calls `getQuizzesByContent`, stores the response array, and resets index/results. It catches errors and always clears loading (`learning.ts:53`).
4. API client sends `GET {baseURL}/quizzes/content/{contentId}` (`api/index.ts:27`).
5. `QuizzesController.findByContentId()` delegates directly to the service (`quizzes.controller.ts:26`).
6. Service first looks up the Content ID. Missing content throws 404 `コンテンツが見つかりません`. It then fetches every quiz for that content, ordered by `sortOrder ASC`, with each choice ordered the same way (`quizzes.service.ts:12`).
7. Choice fields are explicitly limited to `id`, `text`, `sortOrder`. Quiz scalar fields are not limited, so `explanation` and `createdAt` are also returned. `isCorrect` is omitted from choices on this endpoint.
8. `currentQuiz` is `quizzes[currentQuizIndex] || null` (`QuizView.vue:22`). Question and choices use Vue interpolation, not HTML/Markdown rendering (`:108`, `:120`).

There is no pagination, question randomization, choice randomization, or per-question network fetch. All questions arrive together. During loading, the always-rendered counter can show `1 / 0`. A valid content with no quizzes returns `[]` and shows the empty state; this differs from a nonexistent content's 404.

Illustrative response shape, with placeholder values:

```json
[
  {
    "id": "quiz-uuid",
    "contentId": "content-uuid",
    "question": "Question text",
    "explanation": "Explanation text",
    "sortOrder": 1,
    "createdAt": "2026-01-01T00:00:00.000Z",
    "choices": [
      { "id": "choice-uuid", "text": "Choice text", "sortOrder": 1 }
    ]
  }
]
```

## 4. Answer submission and database operations

### Frontend click handling

`selectChoice()` (`QuizView.vue:36`) ignores a click when a result already exists, a request is in flight, or there is no current question. Otherwise it selects the choice, sets `isAnswering=true`, and immediately submits it. There is no separate Submit button.

`learning.answerQuiz()` (`learning.ts:68`) calls the API. On success it finds the quiz in the current array, appends `{ quiz, choiceId, result }` to `quizResults`, and returns the response. The view saves the result and clears `isAnswering`. On failure the store sets its shared error and returns `null`.

The DOM buttons use `:disabled="!!answerResult"`, not `isAnswering`. While waiting they remain visually enabled, but the function guard prevents additional clicks from creating requests in this mounted view. This is a frontend guard, not backend deduplication.

### HTTP contract

```http
POST /quizzes/{quizId}/answer
Authorization: Bearer <JWT>
Content-Type: application/json

{"choiceId":"selected-choice-uuid"}
```

The controller takes the quiz ID from the path and user ID from `req.user`; it does not accept a client-supplied user ID or correctness flag (`quizzes.controller.ts:31`). `SubmitAnswerDto` requires a nonempty string `choiceId`. Global `ValidationPipe` transforms inputs, rejects unknown body fields, and applies DTO validation (`main.ts:28`). No UUID validator is used for path IDs or `choiceId`; whitespace-only strings are not excluded by a trim operation here.

### Ordered service operations

`QuizzesService.submitAnswer()` (`quizzes.service.ts:37`):

1. `quiz.findUnique({ where: { id: quizId }, include: { choices: true } })`.
2. Missing quiz: throw 404 `クイズが見つかりません`.
3. Search the loaded quiz's choices for `selectedChoiceId`. Missing/mismatched choice: throw 400 `選択肢が見つかりません`. This prevents selecting a choice belonging to another quiz.
4. `userQuizAttempt.create()` with `{ userId, quizId, selectedChoiceId, isCorrect: selectedChoice.isCorrect }`. Correctness comes from the database.
5. `quiz.count({ where: { contentId: quiz.contentId } })`.
6. `userQuizAttempt.findMany()` for this user and this content through the quiz relation, selecting distinct `quizId` values across **all historical attempts**.
7. If the number of distinct answered quizzes is at least the current quiz count, upsert `UserProgress` on `(userId, contentId)`, setting `completed=true` and `lastAccessedAt=now`.
8. Find the first loaded choice with `isCorrect=true`, then return:

```json
{
  "isCorrect": false,
  "correctChoiceId": "correct-choice-uuid",
  "explanation": "Explanation text or null",
  "attemptId": "saved-attempt-uuid"
}
```

The controller has no `@HttpCode` override, so Nest's normal POST success status is 201. The frontend uses the body and does not check for a particular success status. `QuizAnswerResult` omits `attemptId` and declares `correctChoiceId` required, although the service can produce `undefined` if no correct choice exists. The optional explanation type also does not explicitly describe backend nulls (`types/index.ts:59`).

Logical Prisma operations in a normal answer: one User lookup for authentication plus four service operations; a fifth service operation runs when completion is satisfied. These are ORM calls, not an exact SQL statement count: relation loading and Prisma's distinct handling can change generated SQL.

There is no transaction around insert/check/completion, no idempotency key, no attempt uniqueness constraint, and no quiz-session table. If an operation after the insert fails, an attempt may persist even though the client sees a failed request. A retry can insert another attempt.

## 5. Feedback, next question, result, and exit

| Action/state | Behavior | Source |
|---|---|---|
| Successful answer | Correct choice becomes green; selected wrong choice becomes red; other choices dim. Feedback shows correctness and explanation. Choices are disabled. | `QuizView.vue:62`, `:111`, `:140` |
| Next question | Increments local index, clears local selected choice/result. No API call. | `QuizView.vue:48`, `learning.ts:82` |
| Last question | Next button label becomes `けっかを見る`; click navigates to `/quiz-result`. No finish request. | `QuizView.vue:49`, `:152` |
| Progress bar | `round(currentQuizIndex / quizzes.length * 100)`. It measures index, not answered count. On the final question it remains below 100%, even after answering. | `QuizView.vue:26` |
| Result score | Counts successful local result entries with `isCorrect=true`; percentage is rounded correct/entries. No backend read or write. | `QuizResultView.vue:10` |
| Result details | Shows question, correctness, and explanation; does not display the selected-choice text. | `QuizResultView.vue:88` |
| Result navigation buttons | Reset quizzes/index/results and navigate to content list or dashboard. Dashboard mount fetches its data again. | `QuizResultView.vue:34`, `DashboardView.vue:13` |
| Quit | Navigates back to the content detail. Does not undo saved answers or explicitly reset the quiz session. The detail mount records progress again. | `QuizView.vue:58`, `ContentDetailView.vue:25` |
| Browser refresh | Pinia quiz state is lost. Refreshing quiz loads from the beginning; refreshing result produces `結果がありません`. Persisted attempts remain in DB. | `main.ts:9`, `QuizView.vue:17`, `QuizResultView.vue:48` |

Results are not persisted to localStorage, URL state, or a backend session record. Only the token is stored in localStorage. Header navigation can also leave a quiz without resetting learning state. Logout clears authentication state but does not clear `learning.quizResults` (`AppHeader.vue:8`, `auth.ts:19`).

## 6. Database model and completion semantics

Source: `sys/backend/prisma/schema.prisma:32`.

```text
Category 1 -> many Content
Content  1 -> many Quiz
Quiz     1 -> many QuizChoice
User     1 -> many UserQuizAttempt -> 1 Quiz
User     1 -> many UserProgress   -> 1 Content
```

- `Quiz`: question, nullable explanation, content ID, sort order, timestamp.
- `QuizChoice`: quiz ID, text, boolean correctness, sort order. The schema does not enforce exactly one correct choice.
- `UserQuizAttempt`: user ID, quiz ID, selected choice ID, correctness snapshot, timestamp. Repeated answers create additional rows. `selectedChoiceId` is a string with no Prisma relation/foreign key to `QuizChoice`; membership is checked in the service.
- `UserProgress`: one row per user/content, enforced with `@@unique([userId, contentId])`; completed flag and timestamps.

Completion means **every current question has been answered at least once in the user's history**, regardless of correctness and regardless of which browser visit supplied the answer. A perfect score is not required. Exiting a quiz partway through leaves previous attempts saved. A later visit can complete the content using those older answers, even before reaching the end of its new frontend session.

Dashboard accuracy instead counts all attempt rows and all correct attempt rows, including repeats (`progress.service.ts:60`). Thus the result page's current-visit score and the dashboard's lifetime accuracy can legitimately differ.

## 7. Failure paths and UI consequences

| Trigger | Backend/client outcome | Visible quiz behavior |
|---|---|---|
| Missing frontend token | Router redirects to login before quiz mount. | No quiz displayed. |
| Invalid/expired token during answer | JWT guard rejects, normally 401. | Store error replaces the entire question/feedback region. No automatic login redirect. |
| Nonexistent content on quiz load | 404 with content-not-found message. | Error view; no choices. |
| Existing content without quizzes | 200 with empty array. | Empty message and Back button. |
| Invalid answer body | 400 validation response; message can be an array. | Error is assigned without normalization; template renders the stored value. |
| Nonexistent quiz / wrong quiz's choice | 404 / 400 respectively. | Error replaces choices and feedback. |
| Network error or unexpected backend failure | Axios rejection; fallback text if no response message. | Same error view. |
| Rate limit exceeded | Global throttler is configured for 300 requests per 60 seconds per IP. | Failed load/answer enters error state. |
| Request never settles | API client specifies no explicit timeout or cancellation. | Load spinner or answer-in-flight state can persist. |

The error branch precedes the question branch (`QuizView.vue:92`). An answer failure therefore hides the choices, although `isAnswering` is cleared; there is no in-place retry button. The shared error is not cleared by `answerQuiz()` or `resetQuizSession()`. A new `fetchQuizzes()` clears it.

`loading` and `error` belong to the entire learning store, rather than to individual requests. There is no abort or stale-response protection. `contentId` is captured once and only `onMounted` loads questions: a same-component route-parameter change has no watcher to reload it. These are source-derived edge cases, not runtime reproductions.

## 8. Content source and deployment path

### Source content to database

`doc/contents/<category>-<order>-<slug>/quiz.json` contains `questions`, each with `question`, `explanation`, and `choices[{text,isCorrect}]`. `content-parser.ts:70` reads it. `seed.ts:60` creates Quiz rows with order `qi+1` and QuizChoice rows with order `ci+1`; generated IDs are UUIDs. Runtime APIs read PostgreSQL, not these JSON files.

A read-only scan found 9 quiz JSON files and 40 questions; each current question has exactly one choice whose `isCorrect` is true. This validates source JSON only, not deployed database contents. The parser and database do not enforce that invariant.

The seed deletes attempts, progress, choices, quizzes, contents, categories, and users before rebuilding (`seed.ts:14`). Development backend Docker startup invokes the seed; production startup runs migrations and the server without an automatic seed (`sys/backend/Dockerfile`, `Dockerfile.prod`). Restarting the development backend through that command can therefore replace learning data and question IDs. No seed was run for this investigation.

### HTTP and database transport

- Development: Vite serves the SPA at port 5173; Axios uses `VITE_API_URL` or `http://localhost:3000`. Vite has no API proxy configured. Nest allows the configured frontend CORS origin, default `http://localhost:5173` (`api/index.ts:4`, `vite.config.ts`, `main.ts:23`). Browser requests with Authorization/JSON can involve CORS preflight.
- Backend: QuizzesModule is imported by AppModule; PrismaService uses `PrismaPg` with `DATABASE_URL` and connects on module initialization (`app.module.ts:19`, `prisma.service.ts:10`). Development Compose connects backend to PostgreSQL service `db` on port 5432.
- Production configuration: frontend build embeds `https://yy-pg.hannari.app` as the API base URL. Traefik sends `/quizzes`, `/contents`, `/progress`, and other API prefixes to Nest; the general host router sends other paths to frontend nginx. nginx has SPA fallback to `index.html` (`docker-compose.prod.yml:52`, `:64`, `:76`; `nginx.conf:17`).

**Production routing overlap, inferred from configuration:** the backend router matches all `PathPrefix('/contents')` requests. SPA routes `/contents`, `/contents/:id`, and `/contents/:id/quiz` share that prefix. A direct browser request or refresh can therefore be sent to Nest before nginx's SPA fallback: `/contents` and `/contents/:id` can produce JSON, and `/contents/:id/quiz` has no matching backend controller route. Client-side navigation after the SPA is loaded avoids an HTML request and can still work. This needs verification against actual deployed Traefik rules; no deployment was contacted.

## 9. Findings to prioritize

| Priority | Finding and evidence | Practical consequence |
|---|---|---|
| High | Public `GET /contents/:id` includes full quiz choices without a field projection (`contents.service.ts:22`, `contents.controller.ts:18`). | Every choice's `isCorrect` is disclosed before answering, including during the ordinary content-page fetch. The separate quiz endpoint's filtering does not prevent this. |
| High | Content mount POSTs progress with no body; controller defaults `completed` to false; upsert overwrites existing completion (`ContentDetailView.vue:25`, `api/index.ts:34`, `progress.controller.ts:40`, `progress.service.ts:42`). | Reopening completed content, or quitting back to it, can set completed=false. Historical attempts remain; a later answer can set it true again. |
| High, configuration-derived | Production frontend and API routes overlap under `/contents` (`docker-compose.prod.yml:52`). | Direct links/refreshes can return API JSON or 404 rather than the SPA. |
| Medium | Answer insert and progress update are separate, nontransactional operations (`quizzes.service.ts:56`). | A failed response may leave a saved answer; retries can duplicate attempts and affect dashboard accuracy. |
| Medium | Answer failure sets the shared error rendered ahead of choices (`learning.ts:76`, `QuizView.vue:97`). | No inline recovery; expired-token answers also leave users on an error screen. |
| Medium | Browser-only result state and logout without learning-store reset (`QuizResultView.vue:10`, `auth.ts:19`). | Result refresh loses the score; in-app account changes can retain prior learning state unless another action resets it. |
| Low | Explanations are included in the public quiz-load response; progress bar is based on index; response type omits attemptId/nullability. | Explanations can be inspected early; bar never reaches 100% on quiz screen; TypeScript does not fully express the wire contract. |

These findings were documented only; no fixes were applied.

## 10. Existing verification coverage and remaining checks

Existing Playwright coverage inspected:

- `sys/e2e/tests/quiz.spec.ts:14`: question/choice/counter rendering.
- `quiz.spec.ts:22`: feedback, one correct icon, disabled choices after submission.
- `quiz.spec.ts:35`: all-question traversal, result route, result row count and summary.
- `sys/e2e/tests/progress.spec.ts:14`: complete a quiz, navigate to dashboard, check attempt total and recent content. It does not assert that completed-content count increased.
- `sys/e2e/tests/security.spec.ts:12`: blocks quiz source JSON through media URLs. It does not check answer disclosure through `/contents/:id`.

Tests use isolated database users and locally signed JWTs (`tests/helpers/auth.ts`), so quiz tests do not exercise Google OAuth. No quiz-specific backend unit tests were found. The backend E2E file is a scaffold expecting `GET /` to return `Hello World!`, not coverage of the quiz flow.

For runtime confirmation, use a populated disposable local environment and check:

1. Network: ordered quiz response, absence of choice correctness on quiz GET, presence on content GET, exact POST body/status/response, and no request on Next/result navigation.
2. Database: one new attempt per normal click; completion after all distinct questions even with wrong answers; history spanning separate visits; repeated answers and dashboard counts.
3. Recovery: expired JWT, request failure after persistence, double click, quit while a request is in flight, refresh of quiz/results, logout and another login.
4. Regression: complete content, reopen it, and inspect `UserProgress.completed`.
5. Deployment: direct GET/refresh of each `/contents` SPA route through the deployed reverse proxy, including expected HTML Content-Type.

No runtime checks above are recorded as passed. They are the concrete follow-up checks needed to validate the source-derived flow against an execution environment.
