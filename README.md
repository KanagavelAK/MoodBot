# 🎵 MoodBot

**Tell a Telegram bot how you feel. Get back a real Spotify playlist built for that mood, in about ten seconds.**

Built with n8n, Google Gemini and the Spotify Web API. Eight nodes, no manual steps.

<img width="1917" height="862" alt="image" src="https://github.com/user-attachments/assets/85210be0-b4d0-42ee-9a83-2b3618893ec1" />


---

## See it work

**You send:**

> I just finished my exams and I feel completely empty

**It replies:**

> Sounds like you're running on empty — here's something quiet to sit with.
>
> 🎵 Your playlist: `https://open.spotify.com/playlist/0AgiZuPRT6nqKYmcBzVP6B`
>
> 10 tracks loaded — enjoy 🎧

The playlist is real. It exists on a Spotify account, it has ten tracks in it, and it did not exist before that sentence was typed.

---

## The design decision

**The AI doesn't pick songs. It produces a search strategy.**

Gemini is good at understanding what a sentence feels like. Spotify is good at finding music. So Gemini translates emotion into a search phrase, and Spotify does the finding. Each system does the thing it's actually good at.

Gemini returns structured JSON:

```json
{
  "mood_label": "drained and hollow",
  "playlist_name": "🌙 Empty Room Echoes",
  "search_query": "melancholic ambient slow reflective",
  "reply": "Sounds like you're running on empty — here's something quiet to sit with."
}
```

There are no keyword lists anywhere in this project. Send it `bittersweet — something ended today but it was good` and you get something reflective, not music that pattern-matched the word "good" to happy.

---

## Flow

```
Telegram message
      ↓
Gemini reads the emotion → returns a search strategy
      ↓
Spotify search → 10 matching tracks
      ↓
Create playlist → add tracks
      ↓
Telegram reply with the link
```

| # | Node | Job |
|---|---|---|
| 1 | User message | Telegram trigger — catches the message, keeps the chat ID as a return address |
| 2 | Mood AI | Sends the text to Gemini, gets back a structured plan |
| 3 | Parse AI | Extracts the JSON from the response, validates it, carries the chat ID forward |
| 4 | Spotify Search | Searches with the AI's phrase, 10 results |
| 5 | Extract URIs | Keeps only the track URIs, drops the rest of the payload |
| 6 | Create Playlist | Creates a real playlist on the authenticated account |
| 7 | Add Items | Fills it with the ten tracks |
| 8 | Send reply | Sends the link back to whoever asked |

The workflow file includes sticky notes explaining each stage on the canvas itself.

---

## Setup

### Prerequisites

- n8n — self-hosted or cloud
- Telegram bot token from [@BotFather](https://t.me/botfather)
- Google AI Studio API key — [aistudio.google.com](https://aistudio.google.com)
- Spotify Developer app — **requires a Spotify Premium account** as of February 2026

### 1. Import

Import `workflow.json` into n8n. All eight nodes arrive wired together; you attach credentials next.

### 2. Credentials

Create three credentials and attach them to the relevant nodes:

| Name | Type | Value |
|---|---|---|
| Telegram bot | Telegram API | Bot token from BotFather |
| Spotify account | Spotify OAuth2 API | Client ID + Secret from the Spotify dashboard |
| AI key | Header Auth | Name: `Authorization` · Value: `Bearer YOUR_GEMINI_KEY` |

Mind the space after `Bearer`.

### 3. Spotify redirect URI

Open the Spotify credential in n8n and copy the **OAuth Redirect URL** shown at the top of the panel. Register that exact string in your Spotify app settings — character for character, no trailing slash.

Running locally, use the loopback IP, because **Spotify rejects `localhost`**:

```
http://127.0.0.1:5678/rest/oauth2-credential/callback
```

Browse n8n on `127.0.0.1` too. Starting the OAuth flow from a `localhost` page sends the callback back on `localhost` and Spotify refuses it.

### 4. Make Telegram reachable

Telegram webhooks require a public HTTPS address, so a local instance needs a tunnel:

```bash
ngrok http 5678
```

Point n8n at the tunnel address:

```bash
set WEBHOOK_URL=https://your-tunnel-address/
n8n start
```

Free tunnel addresses change on every restart, and the workflow must be re-published each time so Telegram registers the new webhook. A reserved ngrok domain, or n8n Cloud, avoids this.

### 5. Adjust for your region

Node 4 uses `market=IN` so results are playable in India. Change it to your own market code, or remove the parameter.

### 6. Publish

Toggle the workflow active and message your bot.

---

## Gotchas

*Accurate as of September 2026. These cost real time to find and aren't well documented anywhere.*

**Spotify removed two playlist endpoints in February 2026.**

| Removed | Replacement |
|---|---|
| `POST /users/{id}/playlists` | `POST /me/playlists` |
| `POST /playlists/{id}/tracks` | `POST /playlists/{id}/items` |

Most tutorials online still use the old ones. The `/me` version infers the account from the auth token, which removes an entire node that existed only to look up your user ID — this workflow is eight nodes where older ones are nine.

**Search `limit` now maxes at 10**, down from 50. The default dropped to 5.

**Gemini 3.x models think before they answer.** They spend tokens on internal reasoning, so a low `max_tokens` gets consumed before the answer starts and the JSON is truncated mid-object. The parser then fails on output that was valid but unfinished. 2000 works; 300 does not. The workflow names this cause explicitly in its error message if it ever happens.

**Spotify rejects `localhost`** in redirect URIs — loopback must be the explicit IP.

**Don't use Telegram's Markdown parse mode for AI-generated text.** One underscore or asterisk in a playlist name makes Telegram reject the entire message — at the final node, after all seven previous ones succeeded.

**Escape user input before it enters a JSON request body.** `JSON.stringify` supplies its own quotes and escapes whatever was typed. Without it, one quotation mark breaks the whole call.

**Model IDs expire.** `gemini-2.5-flash` and `llama-3.3-70b-versatile` were both already unavailable when this was built. If Node 2 returns "model not found", check the current list in AI Studio.

---

## Limitations

Honest ones:

- **All playlists are created on one Spotify account** — whichever authorised the workflow. Users don't need Spotify to open the link, but per-user playlists would require each person to complete their own OAuth flow.
- **Spotify Development Mode** allows one Client ID per developer and five authorised users.
- **Gemini's free tier** throttles at 30 requests per minute.
- **Running locally** means the bot is alive only while the machine and tunnel are.

---

## Possible extensions

- Swap Spotify for the YouTube Data API to remove the Premium requirement
- Add an IF node after the search so an empty result sends a friendly message instead of failing
- Add a separate error workflow with an Error Trigger, assigned under Settings → Error Workflow
- Store past playlists per user so the bot can avoid repeating itself

---

## License

MIT
