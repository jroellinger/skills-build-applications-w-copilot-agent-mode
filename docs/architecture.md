# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- **Presentation tier:** Keep rendering and interaction in the React pages. Use the shared API client for relative `/api` requests, and give each capability an accessible loading, error, empty, and success state. Do not put persistence or business rules in components.
- **Logic tier:** Keep business rules and request validation in the Express service layer; routes pass request data to services and translate results into the API response shape. Invalid or missing input and unknown records receive deliberate 4xx responses, not placeholder responses or cast-error 500s.
- **Data tier:** Use Mongoose models for required fields and relationships, and keep seed data representative, linked, and idempotent. Teams default to an empty members array; activities reference existing users and teams.
- **Activity tracking approach:** Accept `userId`, `teamId`, `type`, and `minutes`; validate both IDs and confirm both records exist before saving. Store the corresponding `user`, `team`, `type`, and `minutes`, calculate one point per minute, and set `occurredAt` when the activity is created.
- **Team management approach:** Require a name, preserve the submitted name and mascot, and store member references. Reject a missing name with `400`.
- **Leaderboard approach:** Sum activity points by team, return `teamName`, `totalPoints`, and a 1-based `rank`, ordered from the largest total to the smallest.
- **Workout suggestions approach:** Find the user by email and prioritize workouts matching their fitness goal; if that does not yield a suggestion, use their latest activity as the fallback focus. Include a reason tied to the goal or activity. Return a deliberate 4xx when no user matches.

## Test plan

Run focused contract tests while implementing each tier, then run both builds. A failed data test points to model or seed behavior; a failed API test points to service, route, or persistence behavior; a failed frontend test points to API consumption, rendering, or accessible UI states. A build failure indicates a TypeScript or bundling problem.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
- API command: `npm run test:api --prefix octofit-tracker/backend`
- Frontend command: `npm test --prefix octofit-tracker/frontend`
- Backend build command: `npm run build --prefix octofit-tracker/backend`
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`

### Initial baseline

Observed on 2026-11-20 before implementation; rerun the commands after changes rather than treating this snapshot as current status:

- Data tests: 4 failed, 1 passed. Failures cover required user identity/goals, team name/mascot, activity fields, and workout fields.
- API tests: 4 failed. Invalid activity and team requests returned `501` instead of `400`; the leaderboard was empty; and a workout suggestion's reason did not match the user's activity pattern.
- Frontend tests: 15 failed. The three top-level API-rendering/request cases failed, as did loading, empty, and API-error states for each of the four capability routes.
- Backend build: passed.
- Frontend build: passed.
