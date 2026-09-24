# Getting this study running

The code in this repo is complete but not connected to anything yet. You need three
free accounts (Firebase, Vercel, OSF/DataPipe) and about an hour. Here's the order
that works best.

## 1. Firebase (the study's database)

This stores participant IDs, condition assignments, and the shared chatbot cache.

1. Go to https://console.firebase.google.com and add a project (any name).
2. In the left menu: Build -> Realtime Database -> Create database. Pick a US region
   and start in **locked mode**. Important: it has to be Realtime Database, not
   Firestore. They look similar in the menu but the code only talks to Realtime.
3. Rules tab: leave it locked (`".read": false, ".write": false`) and publish.
   Looks wrong, is right. Participants never touch the database directly, only the
   server does, and the server uses an admin key that skips the rules.
4. Gear icon -> Project settings -> Service accounts -> Generate new private key.
   A .json file downloads. Treat it like a password.
5. Turn that file into one long line of text. On a Mac:
   `base64 -i ~/Downloads/yourfile.json | pbcopy` (now it's on your clipboard).
6. Copy your database URL from the top of the Data tab. It looks like
   https://yourproject-default-rtdb.firebaseio.com
7. Paste that URL into the one spot in `index.html` marked with the placeholder
   (search for YOUR-PROJECT to find it).

You don't create any tables. They appear on their own the first time the app writes.

## 2. OSF + DataPipe (where the final data files land)

1. Make an account at https://osf.io and create a private project for the study.
2. Make an account at https://pipe.jspsych.org (DataPipe) and connect it to OSF
   when it asks.
3. In DataPipe, create an experiment linked to your OSF project. Copy its
   experiment ID (short code like aB3xY9zQwK).
4. Create a **second** experiment for consent PDFs, also linked to your OSF project,
   and turn on "Enable base64 data collection" on its dashboard. Copy that ID too.
5. In `index.html`, search for REPLACE_WITH and paste the first ID over
   REPLACE_WITH_YOUR_DATAPIPE_ID and the second over REPLACE_WITH_CONSENT_DATAPIPE_ID.

Until you do step 5 the app still runs, it just keeps completed sessions in the
browser's localStorage instead of uploading them.

## 3. Vercel (puts it on the internet)

1. Make an account at https://vercel.com with "Continue with GitHub".
2. Add New -> Project -> import this repo. Don't change any build settings.
3. Before hitting Deploy, add these environment variables:

   - `OPENAI_API_KEY` - from https://platform.openai.com (API keys page, needs
     billing set up, a few dollars is plenty)
   - `FIREBASE_SERVICE_ACCOUNT_BASE64` - the long line from Firebase step 5
   - `FIREBASE_DATABASE_URL` - the URL from Firebase step 6
   - `TOKEN_SECRET` - any random string, 32+ characters, you never need to
     remember it

   There are also a few optional ones you probably won't need: `ALLOWED_ORIGINS`
   (only if another domain needs to call the API), `PER_IP_TOKEN_LIMIT` (default
   15 tokens per IP per hour) and `PER_TOKEN_CALL_LIMIT` (default 500 calls per
   session).

4. Deploy. You get a live URL like https://something.vercel.app after a minute.
   Every git push after this redeploys automatically.

## 4. Check it works

Open your URL and run through the whole study once like a participant. Then check:
your Firebase Data tab should show a participant counter and session data, and your
OSF project should have a JSON data file and a consent PDF in it. If both are there,
you're done.

Tip: add `?researcher=hpl-staff` to the URL to get the researcher panel for faster
clicking around while testing. Before collecting real data, delete your test entries
from the Firebase Data tab (hover a node, three dots, delete) and the test files
from OSF.
