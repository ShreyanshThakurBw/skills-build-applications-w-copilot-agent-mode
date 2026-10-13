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

- Presentation tier decision: Build a small React route-based UI in `octofit-tracker/frontend/src` that fetches the real `/api` responses, renders the relevant data for activities, teams, leaderboard, and workout suggestions, and includes the required loading, empty, and error states with accessible text and roles.
- Logic tier decision: Keep the Express API thin and contract-driven: validate request bodies, resolve user/team references with explicit 4xx handling, compute derived values in services, and expose JSON shapes that match the API contract exactly without leaking database errors.
- Data tier decision: Use Mongoose models as the source of truth for validation and seeded fixtures. Keep relationships between `User`, `Team`, and `Activity` explicit, preserve unrelated records during seeding, and ensure the seed script is idempotent so repeated runs do not duplicate or overwrite prior data.
- Activity tracking approach: Accept `userId`, `teamId`, `type`, and `minutes`; confirm both IDs refer to existing records; reject malformed IDs and missing references with deliberate 4xx responses; compute `points` and `occurredAt` on create; store the normalized `user`, `team`, `type`, and `minutes` values.
- Team management approach: Require `name` and `mascot`, default `members` to an empty array, and allow member IDs to be stored as ObjectIds while exposing them to the API as strings for consumers.
- Leaderboard approach: Aggregate team activity totals from `Activity` records, sum `points` by team, sort descending, and assign a 1-based rank so the API returns stable, ordered results for the UI.
- Workout suggestions approach: Look up the matching user by email, inspect recent activity patterns or fitness goal, choose the most relevant workout by `focus`, and return a `reason` that ties the recommendation back to the user’s recent performance or objective.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` — validates the Mongoose contract. Current failures mean the schemas do not yet require the fields and relationships the API needs (`name`, `email`, `fitnessGoal`, team membership, activity metadata, and workout focus/duration/difficulty).
- API command: `npm run test:api --prefix octofit-tracker/backend` — validates the Express `/api/*` contract. Current failures mean the routes are not yet returning the expected 4xx validation responses, cannot create valid records, leave the leaderboard empty, or fail to match workout suggestions to user activity.
- Frontend command: `npm test --prefix octofit-tracker/frontend` — validates the presentation contract. Current failures mean the app still renders placeholder “Implementation pending” content instead of loading real API data and accessible states.
- Backend build command: `npm run build --prefix octofit-tracker/backend` — confirms the TypeScript backend compiles after each tier fix. This catches route/service/type issues before moving on to UI-level validation.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` — confirms the React app compiles after data and API work are complete. This is the last safety check for presentation wiring and route rendering.
