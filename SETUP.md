# Getting this study running

The code in this repo is complete but not connected to anything yet. You need three
free accounts (Firebase, Vercel, DataPipe/Dataverse), an OpenAI API key with a few dollars
loaded onto it, and about an hour of time. Here's the order that works best.

## 1. Create a temporary local file for storing keys

Create an Excel file with the following values in the first column:
   - `FIREBASE_SERVICE_ACCOUNT_BASE64`
   - `FIREBASE_DATABASE_URL`
   - `DATAVERSE_API`
   - `OPENAI_API_KEY`
   - `TOKEN_SECRET`

This is just for keeping track of these values temporarily as they're generated.
DO NOT EVER upload / share these values anywhere else.

## 2. Firebase (the study's database)

This stores participant IDs, condition assignments, and the shared chatbot cache.

1. Go to https://console.firebase.google.com and add a project (any name).
2. In the left menu: Build -> Realtime Database -> Create database. Pick a US region
   and start in **locked mode**. Important: it has to be Realtime Database, not
   Firestore. They look similar in the menu but the code only talks to Realtime.
3. In the rules tab:

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "email_hashes": {
      "$hash": {
        ".read": true,
        ".write": false
      }
    }
  }
}
```
   Then publish.
   
4. Gear icon -> Project settings -> Service accounts -> Generate new private key.
   This will download a .json file. 
5. Turn that file into one long line of text. On a Mac:
   `base64 -i ~/Downloads/yourfile.json | pbcopy` (this "converts" the entire contents 
   of the .json file into a single string, and saves it to your clipboard.) 
   **Paste this into your Excel spreadsheet next to `FIREBASE_SERVICE_ACCOUNT_BASE64`.**
6. Copy your database URL from the top of the Data tab. It looks like
   https://yourproject-default-rtdb.firebaseio.com. **Paste this into your Excel**
   **spreadsheet next to `FIREBASE_DATABASE_URL`.** 
   NOTE: Make sure the URL DOES NOT have a "/" at the end.

## 3. Dataverse (where the final data files land)

1. Make an account at https://dataverse.harvard.edu/ (Harvard Dataverse).
2. Create a "Dataverse" (Add Data > New Dataverse)
      - `HOST DATAVERSE`: Keep as "Harvard Dataverse"
      - `DATAVERSE NAME`: Whatever you want the title of this research project to be
        (e.g., "Lonnie's HPL Study Clone")
      - `DATAVERSE_COLLECTION_ALIAS`: The "short-title" that gets appended to the end of
        this Datavers's URL. NOTE: You will need this when setting up your DataPipe experiments
        in the next step
      - `CATEGORY`: Research Project
   Everything else is optional and can be left blank or as-is.

3. Create an API Token on Dataverse (click your username the top-right > API Token > Create Token). **Paste this into your Excel spreadsheet next to `DATAVERSE_API`.**

## 4. DataPipe (the intermediary between the study and Dataverse)

1. Make an account at https://pipe.jspsych.org (DataPipe).
2. Make an account at https://dataverse.harvard.edu/ (Harvard Dataverse).
3. On Dataverse
4. In DataPipe, create an experiment linked to your OSF project. Copy its
   experiment ID (short code like aB3xY9zQwK).
5. Create a **second** experiment for consent PDFs, also linked to your OSF project,
   and turn on "Enable base64 data collection" on its dashboard. Copy that ID too.
6. In `index.html`, search for REPLACE_WITH and paste the first ID over
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
