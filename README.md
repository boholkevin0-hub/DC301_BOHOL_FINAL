# Campusdesk — Student Document Portal

A responsive single-page Vue application for creating a student account, requesting official school documents, and tracking their progress online.

## Run locally

```sh
npm install
npm run dev
```

Create a production build with `npm run build`.

## Pages

- **Create account / sign in (`/`)** — Create an account with your name, student ID, email, program, year level, and password. Sign in with the account afterward.
- **Dashboard (`/`)** — Request summary, latest updates, and a shortcut to submit a request.
- **New request (`/request`)** — Create a request for a Certificate of Enrollment, Good Moral Certificate, Transcript of Records, Certificate of Grades, or another school document. Required fields are marked.
- **My requests (`/requests`)** — Search, filter, read, edit, and delete requests; see the request status and release estimate.
- **My profile (`/profile`)** — Enter and update your student contact and program information.

The client-side router uses the browser History API. Accounts, profile details, and requests are stored in the current browser's local storage. Passwords are stored as salted PBKDF2 hashes, but this client-only account system is not a substitute for server-side authentication and cannot be used to securely share accounts or data across devices. Connect a trusted authentication service and backend before using this app with real student accounts.

## Requirements covered

- Vue 3 single-page application built with Vite.
- Navigable pages with browser back/forward support.
- Reactive components, form bindings, validation, computed filtering, and status summaries.
- CRUD operations for document requests.
- Responsive layouts and accessible form labels and action controls.
