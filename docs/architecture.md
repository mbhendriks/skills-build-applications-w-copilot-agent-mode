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

- Presentation tier decision: Keep page rendering and interaction in React components, with all HTTP URL construction and fetch behavior in the shared API client. Each capability shows accessible loading, empty, success, and error states; components do not access MongoDB or contain business rules.
- Logic tier decision: Keep request parsing, input validation, and HTTP status mapping in Express routes; put business behavior in explicit service functions. Invalid input and missing records return deliberate 4xx responses, and authentication remains out of scope.
- Data tier decision: Enforce required fields and references in Mongoose schemas, and use the existing shared database configuration. Seed stable demo records with upserts so repeated runs are safe and do not clear user data.
- Activity tracking approach: Accept `userId`, `teamId`, `type`, and positive `minutes`; validate ID format and confirm both records exist before writing. Store the user/team references, type, minutes, computed points (one point per minute), and occurrence time; return validation failures as 4xx responses.
- Team management approach: Require a non-empty name, retain the submitted name and mascot, and store submitted member IDs as team references. Return created and listed teams with their members.
- Leaderboard approach: Sum persisted activity points by team, order totals highest first, and return each team's name, total points, and 1-based rank.
- Workout suggestions approach: Find the user by email, use the user's goal and recent activity to rank stored workouts, and include a reason with the first suggestion. Return a deliberate 4xx response when no user matches.

Implement in dependency order: data schemas and their contract tests; backend services/routes and API contracts; frontend API-backed views and UI states; then run both builds.

## Test plan

Run these checks in order as implementation progresses; a failure should be diagnosed in the named tier before moving on.

| Check | What a failure indicates |
| --- | --- |
| `npm run test:data --prefix octofit-tracker/backend` | Mongoose schema constraints, references, or seed idempotency do not meet the data contract. |
| `npm run test:api --prefix octofit-tracker/backend` | Route validation/status handling or service behavior does not meet the API contract, including activity creation, teams, leaderboard, or workout suggestions. |
| `npm test --prefix octofit-tracker/frontend` | A view is not loading API data or does not provide the required accessible loading, empty, success, or error state. |
| `npm run build --prefix octofit-tracker/backend` | Backend TypeScript has a compile-time error. |
| `npm run build --prefix octofit-tracker/frontend` | Frontend TypeScript or Vite production build has an error. |
