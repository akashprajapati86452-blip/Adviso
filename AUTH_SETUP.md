# Adviso — Firebase Auth Setup

This adds fully functional **Email/Password + Google Sign-In**, session
management, and per-user Firestore data to the Adviso dashboard. Follow
these steps in order — none of the auth code will work until step 1–3 are
done, because there's no such thing as "demo" Firebase Auth: it always
talks to a real project.

## 1. Create a Firebase project
1. Go to https://console.firebase.google.com → **Add project**.
2. Once created, click the **Web (`</>`)** icon to register a web app.
3. Copy the `firebaseConfig` values shown — you'll need them for `.env.local`.

## 2. Enable sign-in providers
Firebase Console → **Authentication → Sign-in method**:
- Enable **Email/Password**.
- Enable **Google**. Set a support email when prompted.
- Under **Settings → Authorized domains**, add your production domain
  (e.g. `app.adviso.ai`) once you deploy — `localhost` is allowed by default.

## 3. Create the Firestore database
Firebase Console → **Firestore Database → Create database** → start in
**production mode** (the security rules in `firestore.rules` handle access
control — you do not need "test mode").

Deploy the rules:
```bash
npm install -g firebase-tools
firebase login
firebase init firestore   # point it at this project, keep the existing firestore.rules
firebase deploy --only firestore:rules
```

## 4. Configure environment variables
```bash
cp .env.local.example .env.local
```
Paste in the 6 values from step 1. Restart `npm run dev` after editing
`.env.local` — Next.js only reads it on startup.

## 5. Install dependencies and run
```bash
npm install
npm run dev
```
Visit `/signup`, create an account, and you should land on `/dashboard`
with your name and a starter task list. Visit `/login` in an incognito
window to confirm session persistence and logout both work.

## How the pieces fit together
| File | Responsibility |
|---|---|
| `lib/firebase.js` | Initializes the Firebase app once, exports `auth`/`db`/`googleProvider` |
| `lib/AuthContext.jsx` | The one place that calls Firebase Auth — signup, login, Google, logout, password reset/change |
| `lib/firestoreUser.js` | Creates/reads the `users/{uid}` profile doc, seeds starter data for new accounts |
| `hooks/useUserData.js` | Real-time (`onSnapshot`) hooks for a user's tasks and notifications |
| `components/auth/*` | Login, Signup, Forgot Password forms + `ProtectedRoute` guard |
| `components/dashboard/DashboardShell.jsx` | The dashboard UI, now reading the real signed-in user everywhere it used to say "Ramesh Gupta" |
| `app/*/page.jsx` | Next.js App Router routes wiring the above together |

## What's real vs. still sample data
Fully live per authenticated user, backed by Firestore:
- Identity (name, email, business name) everywhere in the UI
- Daily Action Plan tasks (add / check off / persists per user)
- Notifications (mark-as-read persists per user)
- Settings (profile edits, password change, logout)

Still representative **sample** data (same for every account, not yet
wired to a backend) — clearly isolated as constants at the top of
`DashboardShell.jsx` so you can swap them for real queries once you have
a data source (POS integration, CSV import, etc.):
- Sales Analytics, Revenue, Customer Analytics, Marketing Insights charts
- The Tasks kanban board and Reports list

## Security notes
- The `NEXT_PUBLIC_FIREBASE_*` values are meant to be public — Firebase
  web apps are secured by **Firestore Security Rules**
  (`firestore.rules`), not by hiding the config. Don't skip deploying
  the rules file.
- Changing a password requires re-authentication
  (`reauthenticateWithCredential`) — this is Firebase enforcing that a
  stale session can't silently change account security, not a bug.
- For account deletion, don't wire it straight to the client SDK; route
  it through a Cloud Function so you can clean up the user's
  subcollections and any billing records atomically.

## Production checklist before launch
- [ ] Firestore rules deployed (step 3) and tested with the Firebase Rules Playground
- [ ] Google provider's OAuth consent screen configured with your real app name/logo (Google Cloud Console)
- [ ] Custom email templates for password reset / email verification (Firebase Console → Authentication → Templates)
- [ ] Rate limiting / App Check enabled if you're worried about abuse (Firebase Console → App Check)
- [ ] Move the Adviso AI Chat's Anthropic API call behind your own backend route instead of calling it directly from the browser (see note in the previous dashboard delivery)
