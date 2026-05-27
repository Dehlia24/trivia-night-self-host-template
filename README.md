# Trivia Night

A real-time multiplayer trivia game. No backend or server required — just three HTML files, a free Firebase project, and somewhere to host them.

Players join from their phones using a 6-character game code. The host controls the game from a dashboard. Scores update live as answers come in.

---

## Table of Contents

1. [What you need](#1-what-you-need)
2. [Step 1 — Set up Firebase](#2-step-1--set-up-firebase)
3. [Step 2 — Add your Firebase config to the files](#3-step-2--add-your-firebase-config-to-the-files)
4. [Step 3 — Set up Firebase Security Rules](#4-step-3--set-up-firebase-security-rules)
5. [Step 4 — Choose how to host](#5-step-4--choose-how-to-host)
   - [Option A — GitHub Pages (free, public)](#option-a--github-pages-free-public)
   - [Option B — Local network (LAN party / same Wi-Fi)](#option-b--local-network-lan-party--same-wi-fi)
6. [Step 5 — Set up the Session Manager (optional)](#6-step-5--set-up-the-session-manager-optional)
7. [How to run a game](#7-how-to-run-a-game)
8. [Player experience](#8-player-experience)

---

## 1. What you need

- A free [Google account](https://accounts.google.com) (for Firebase)
- A free [GitHub account](https://github.com) (only needed for GitHub Pages hosting)
- A text editor (Notepad, VS Code, etc.)
- That's it — no Node.js, no npm, no build tools

---

## 2. Step 1 — Set up Firebase

Firebase is the free real-time database that syncs game state between the host and all players.

1. Go to [console.firebase.google.com](https://console.firebase.google.com) and sign in with your Google account.
2. Click **Add project**, give it any name (e.g. `my-trivia-night`), and click through the setup steps. You can turn off Google Analytics when prompted — it's not needed.
3. Once the project is created, click **Continue** to open the project dashboard.
4. In the left sidebar, click **Build** → **Realtime Database**.
5. Click **Create Database**.
6. Choose a location (any region is fine) and click **Next**.
7. Select **Start in test mode** and click **Enable**.
   > Test mode allows all reads and writes for 30 days. You will set proper rules in Step 4 before that window expires.
8. Now get your config values:
   - Click the gear icon next to **Project Overview** in the left sidebar → **Project settings**.
   - Scroll down to the **Your apps** section.
   - Click the **</>** (web) icon to register a web app. Give it any name and click **Register app**.
   - You will see a `firebaseConfig` object that looks like this:
     ```js
     const firebaseConfig = {
       apiKey: "AIza...",
       authDomain: "your-project.firebaseapp.com",
       databaseURL: "https://your-project-default-rtdb.firebaseio.com",
       projectId: "your-project",
       storageBucket: "your-project.firebasestorage.app",
       messagingSenderId: "123456789",
       appId: "1:123456789:web:abc123"
     };
     ```
   - Keep this tab open — you will need these values in the next step.

---

## 3. Step 2 — Add your Firebase config to the files

You need to paste your config into three files. Open each file in a text editor and find the `firebaseConfig` block (it is near the top of the `<script>` section). Replace every placeholder value with your real values from the Firebase console.

**Files to edit:**
- `trivia_night_host_dashboard.html`
- `trivia_night_player.html`
- `trivia_night_games.html`

In each file, find this block and replace it:

```js
const firebaseConfig = {
  apiKey:            "YOUR_API_KEY",
  authDomain:        "YOUR_PROJECT_ID.firebaseapp.com",
  databaseURL:       "https://YOUR_PROJECT_ID-default-rtdb.firebaseio.com",
  projectId:         "YOUR_PROJECT_ID",
  storageBucket:     "YOUR_PROJECT_ID.firebasestorage.app",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId:             "YOUR_APP_ID"
};
```

Replace each `YOUR_...` placeholder with the matching value from your Firebase project. All three files must have the same config — they all talk to the same database.

> **Note on API keys:** The Firebase `apiKey` is safe to put in a public GitHub repo. It is not a secret — it just identifies your project. Security is handled by Firebase Security Rules (which you will set up next), not by hiding the key.

---

## 4. Step 3 — Set up Firebase Security Rules

The default test mode rules expire after 30 days and then lock out all users. Follow these steps to set permanent rules that keep the game working.

1. Go to [console.firebase.google.com](https://console.firebase.google.com) and open your project.
2. In the left sidebar, click **Build** → **Realtime Database**.
3. Click the **Rules** tab at the top.
4. Replace everything in the editor with the following:

```json
{
  "rules": {
    "games": {
      ".read": true,
      ".write": true
    },
    "templates": {
      ".read": true,
      ".write": true
    }
  }
}
```

5. Click **Publish**.

These rules allow full access to the `/games` and `/templates` paths (which the app uses) and block everything else. Since this app has no login system, the game code is the only access control for active games.

---

## 5. Step 4 — Choose how to host

You have two main options depending on your situation.

---

### Option A — GitHub Pages (free, public)

Best for: hosting a permanent public URL that anyone on the internet can reach.

1. Make sure you have a [GitHub account](https://github.com).
2. Create a new repository:
   - Go to [github.com/new](https://github.com/new)
   - Give it a name (e.g. `trivia-night`)
   - Set it to **Public**
   - Click **Create repository**
3. Upload the three HTML files to the repo:
   - Click **uploading an existing file** on the repo page
   - Drag and drop all three HTML files
   - Click **Commit changes**
4. Enable GitHub Pages:
   - Go to your repo → **Settings** → **Pages** (in the left sidebar)
   - Under **Source**, select **Deploy from a branch**
   - Set Branch to `main` and folder to `/ (root)`
   - Click **Save**
5. Wait about 1-2 minutes, then your site will be live at:
   ```
   https://YOUR_GITHUB_USERNAME.github.io/trivia-night/
   ```
6. Your URLs will be:
   - **Host dashboard:** `https://YOUR_GITHUB_USERNAME.github.io/trivia-night/trivia_night_host_dashboard.html`
   - **Player page:** `https://YOUR_GITHUB_USERNAME.github.io/trivia-night/trivia_night_player.html`
   - **Session manager:** `https://YOUR_GITHUB_USERNAME.github.io/trivia-night/trivia_night_games.html`
7. Add your GitHub Pages domain to Firebase's authorized list:
   - In Firebase Console → **Authentication** → **Settings** → **Authorized domains**
   - Click **Add domain** and enter `YOUR_GITHUB_USERNAME.github.io`
   - Click **Add**

---

### Option B — Local network (LAN party / same Wi-Fi)

Best for: running a game at a party or event where everyone is on the same Wi-Fi network. No internet required after setup.

**What you need:** Python (comes pre-installed on Mac and most Linux systems; download from [python.org](https://python.org) on Windows).

1. Open a terminal (Mac: Terminal app; Windows: Command Prompt or PowerShell; Linux: your terminal).
2. Navigate to the folder where you saved the HTML files:
   ```
   cd path/to/your/trivia-night-folder
   ```
3. Start a local web server:
   - **Mac / Linux:**
     ```
     python3 -m http.server 8080
     ```
   - **Windows:**
     ```
     python -m http.server 8080
     ```
4. Find your local IP address:
   - **Mac:** System Settings → Wi-Fi → Details → IP Address (looks like `192.168.x.x`)
   - **Windows:** Run `ipconfig` in Command Prompt, look for `IPv4 Address`
   - **Linux:** Run `ip addr` in terminal, look for `inet` under your Wi-Fi interface
5. Your URLs are:
   - **Host dashboard:** `http://YOUR_LOCAL_IP:8080/trivia_night_host_dashboard.html`
   - **Player page:** `http://YOUR_LOCAL_IP:8080/trivia_night_player.html`

   For example: `http://192.168.1.42:8080/trivia_night_host_dashboard.html`

6. Open the host dashboard on your computer. Share the player URL with everyone else on the same Wi-Fi (or show it as a QR code — the host dashboard generates one automatically).
7. To stop the server when you're done, press `Ctrl + C` in the terminal.

> **Firewall note:** If players can't connect, your system firewall may be blocking port 8080. On Windows you may see a prompt asking to allow access — click Allow. On Mac you may need to go to System Settings → Network → Firewall and temporarily allow incoming connections, or just allow the Python app when prompted.

> **Firebase still needs internet:** The game state syncs through Firebase, so everyone (host and players) needs internet access even on a local setup. The local server just serves the HTML files — Firebase handles the real-time sync.

---

## 6. Step 5 — Set up the Session Manager (optional)

The Session Manager (`trivia_night_games.html`) is an admin page that shows all active and recent games in real time. It requires a GitHub login and is restricted to a single GitHub username that you specify.

**To set it up:**

1. Open `trivia_night_games.html` in a text editor and find this line:
   ```js
   const ALLOWED_GITHUB_LOGIN = 'YOUR_GITHUB_USERNAME';
   ```
   Replace `YOUR_GITHUB_USERNAME` with your actual GitHub username (case-sensitive).

2. Enable GitHub login in Firebase:
   - Go to Firebase Console → **Authentication** → **Sign-in method**
   - Click **GitHub** and toggle it to **Enabled**
   - You will need a GitHub OAuth App to get a Client ID and Secret (next step)

3. Create a GitHub OAuth App:
   - Go to [github.com/settings/developers](https://github.com/settings/developers) → **OAuth Apps** → **New OAuth App**
   - **Application name:** anything you like
   - **Homepage URL:** your GitHub Pages URL (or `http://localhost:8080` for local)
   - **Authorization callback URL:** `https://YOUR_PROJECT_ID.firebaseapp.com/__/auth/handler`
     (replace `YOUR_PROJECT_ID` with your actual Firebase project ID)
   - Click **Register application**
   - Copy the **Client ID** and generate a **Client Secret**

4. Back in Firebase Console → Authentication → GitHub:
   - Paste in the Client ID and Client Secret
   - Click **Save**

5. Add your hosting domain to Firebase's authorized list:
   - Firebase Console → **Authentication** → **Settings** → **Authorized domains**
   - Add `YOUR_GITHUB_USERNAME.github.io` (for GitHub Pages) or `localhost` (for local)

Once set up, only the GitHub account matching `ALLOWED_GITHUB_LOGIN` can sign in to the Session Manager. Anyone else sees an Access Denied screen.

---

## 7. How to run a game

1. **Open the host dashboard** in your browser. A unique 6-character game code is automatically generated.

2. **Share the player URL** with your players — they can:
   - Scan the QR code shown on the waiting screen
   - Click the player link displayed on screen
   - Go to the player page directly and type the game code

3. **Build your question bank** using the panel on the right side of the host dashboard:
   - Click **+ Add Question** to add a question (new questions appear at the top)
   - Fill in the question text, answer, category, and point value
   - Use the **Pin** button to guarantee a question appears in the next game regardless of whether it has been used before
   - Click **Clear pins** to unpin all questions

4. **Use templates (optional):** Save your question bank as a named template to reuse across sessions. Load a template from the dropdown to restore a saved set.

5. **Set up the game:**
   - Enter a game name in the waiting screen
   - Choose how many questions to include per round
   - Watch players appear in the lobby as they join
   - Use **Kick** to remove a player before starting

6. **Start the game** by clicking **Start Game**. Questions are served one at a time.

7. **Advance questions** by clicking **Next Question**. The host dashboard shows all player responses and running scores in real time.

8. **End the game:** When all questions are done, a final leaderboard is shown to everyone. From that screen you can click **Play Again** (new code, fresh game) or **End Session** (cleans up the session data).

---

## 8. Player experience

1. Open the player URL or scan the QR code from the waiting screen
2. Enter a name and tap **Join Game**
3. Wait in the lobby until the host starts
4. Answer each question before the timer runs out
5. See your running score after each question and the final leaderboard at the end

**Rejoin support:** If a player accidentally closes the tab, reopening the player URL will detect their existing session and offer to rejoin with their score intact.

---

## Tech stack

| Layer | Tool |
|---|---|
| Hosting | GitHub Pages or any static file server |
| Database | Firebase Realtime Database (free tier) |
| Auth (Session Manager only) | Firebase Authentication + GitHub OAuth |
| QR codes | qrcodejs (runs entirely in the browser) |

## Contributing

Have a great set of trivia questions? [Contribute a question bank](CONTRIBUTING.md) to the community library.

---

## About

This project was built and is maintained using [Claude Code](https://claude.ai/code) by Anthropic.
