# Getting this study running

The code in this repo is complete but not connected to anything yet. You need three
free accounts (Firebase, Vercel, DataPipe/Dataverse), an OpenAI API key with a few dollars
loaded onto it, and about an hour of time. Here's the order that works best.

First, copy this GitHub template (Use this template > Create a new repository). Then follow the steps below.

## 1. Create a temporary local file for storing keys

Create an Excel file with the following values in the first column:
   - `FIREBASE_SERVICE_ACCOUNT_BASE64`
   - `FIREBASE_DATABASE_URL`
   - `OPENAI_API_KEY`
   - `TOKEN_SECRET`
   - `DATAVERSE_API_TOKEN`
   - `DATAPIPE MAIN EXPERIMENT ID`
   - `DATAPIPE CONSENT EXPERIMENT ID`
   - `DATAPIPE XLSX EXPERIMENT ID`
   - `DATAPIPE EMAIL EXPERIMENT ID`
   
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
   
4. Download a private key for the project (Gear icon > Project settings > Service accounts > Generate new private key).
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
      - **Host Dataverse**: Keep as "Harvard Dataverse"
      - **Dataverse Name**: Whatever you want the title of this research project to be
        (e.g., "Lonnie's HPL Study Clone")
      - **Identifier**: The "short-title" that gets appended to the end of
        this Datavers's URL (e.g., "hpl-study-lb"). NOTE: You will need this when
        setting up your DataPipe experiments in the next step.
      - **Category**: Research Project
   Everything else is optional and can be left blank or as-is.

3. Create an API Token on Dataverse (click your username the top-right > API Token > Create Token). **Paste this into your Excel spreadsheet next to `DATAVERSE_API_TOKEN`.**

## 4. DataPipe (the intermediary between the study and Dataverse)

1. Make an account at https://pipe.jspsych.org (DataPipe), using
   whatever sign-in method you prefer (email, Google, GitHub, etc.).
3. Link your DataPipe account to Dataverse (Account > Settings > Dataverse > Connect)
     - **Dataverse Server URL***: https://dataverse.harvard.edu/
     - **API Token**: The value you pasted in your Excel spreadsheet under `DATAVERSE_API_TOKEN`.
       (Can also be re-accessed on your Dataverse account, though you should not need this
       token again after this point.)
4. In DataPipe, create **FOUR unique experiments**:
      1.  **Main**: For the primary .json output from the study.
      2.  **Consents**: For consent form PDFs.
      3.  **Emails**: For email .json files, to prevent duplicate participants.
      4.  **XLSX**: For the .xlsx files of each participant's data.

   Create each experiment in the following way:
     - **Title**: An informative study title (e.g., "Lonnie's HPL Study Clone - Consents").
       NOTE: When you create an experiment in DataPipe, it will automatically create a
       collection in Dataverse with the same title that you set here.
     - **Collection alias**: The "Identifier" you chose when setting up your
       Dataverse in the previous step.

   Everything else should be self-explanatory (e.g. name, email, etc.).

5. Configure the four DataPipe experiments:
     - For your "Main" experiment:
         - Turn on "Accept new data"
         - Turn on "Validation"
         - Check "Allow JSON" and "Allow CSV"
         - Make sure the "Required fields" field is empty
     - For your other three experiments:
         - Turn on "Accept new data"
         - Turn on "Accept base64 file uploads"
         - Turn OFF "Validation"
      
   No other changes should be required here.

6. Once all four experiments are created, locate their experiment IDs (under "Experiment details"
   for each experiment -- should be a short code like hz6pAcxZd4Xd).
   **Paste each experiment ID into your Excel spreadsheet next to the corresponding `EXPERIMENT ID` label.**

## 5. OpenAI (enables chatbot access)

1. Log in or make an account at https://platform.openai.com
2. In the side menu, click API Keys, then Create New Secret Key in the top-right. Name, project, and expiration are up to you; make sure Permissions are set to All. NOTE: You'll need to have billing set up and at least a few dollars added to your account.
3. Click Create Secret Key. **Paste this into your Excel spreadsheet next to `OPENAI_API_KEY`.** NOTE: Once you close this window, you WILL NOT be able to view this secret key again. 

## 6. Edit `index.html` (connects this repo to your study-specific information)

1. Open index.html

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
