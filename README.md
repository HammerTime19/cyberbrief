# CyberBrief

Cybersecurity news as 60-word swipeable cards, Inshorts-style, with push alerts for critical stories.

Every 30 minutes a GitHub Action pulls 7 security news feeds plus the CISA KEV catalog, drops ads and webinars, and sends new stories to an AI model. The model writes a headline, a 60-word summary, a severity (critical, high, info) and a category for each one, and merges duplicates when several outlets cover the same incident. The cards are published to a phone-installable web app on GitHub Pages. Critical stories, and high-severity stories that match your watchlist, are pushed to your phone through ntfy. A morning summary arrives at 7am.

Cost: $0. By default the summaries use GitHub Models' free tier through the token GitHub Actions already provides, so no AI account or key is needed. Hosting, scheduling and notifications are free on a public repo. The app's bottom corner shows how many AI calls you've used this month.

| AI provider | Cost | Setup | Free limit (approx.) | Typical use |
|---|---|---|---|---|
| `github` (default) | Free | None | ~150 requests/day | 50–100/day |
| `gemini` | Free | Google AI Studio key | ~1,000 requests/day | 50–100/day |
| `anthropic` | ~$3–5/month | Anthropic key + credit | None | 50–100/day |

Free-tier limits are set by GitHub and Google and can change. If a run hits a limit, the stories it couldn't summarize are retried on the next run.

## Setup (about 15 minutes)

1. **Create the repo.** Make a new public GitHub repo (for example `cyberbrief`) and upload every file from this folder, including the hidden `.github` folder. The default branch must be `main`.

2. **Set up phone notifications.** Install the ntfy app (iOS or Android). Tap + and subscribe to a topic name nobody could guess, such as `cyberbrief-7f3k9q2m8x`. Topics on ntfy.sh are public to anyone who knows the name, so treat it like a password.

3. **Add the secret.** In the repo, go to Settings > Secrets and variables > Actions and add a repository secret named `NTFY_TOPIC` with your topic name.

4. **Turn on Pages.** Go to Settings > Pages and set Source to "GitHub Actions".

5. **Run it.** Go to the Actions tab, choose "Update CyberBrief", then Run workflow. The first run summarizes the 15 newest stories and sends no alerts, so you don't get a flood of notifications.

6. **Install the app.** Open `https://<your-username>.github.io/<repo-name>/` on your phone. On iPhone, use Share > Add to Home Screen in Safari. On Android, use Chrome's menu > Add to Home screen.

## Using it

- Swipe up for the next story. Tap a filter to see only critical stories or one category.
- Tap CyberBrief at the top to jump back to the newest story and refresh.
- "New" marks stories added since you last closed the app.
- Tapping a notification opens the app on that story.
- On a computer, use the arrow keys or j/k.

## Weekly recap, CVE details and owner alerts

- **Weekly recap:** every Sunday at 6pm (`RECAP_WEEKDAY`, `RECAP_HOUR`), the AI writes a "Week in security" card with 5 bullets from the week's critical and high stories, each linking to its card. To make one now, run the workflow with "Write a weekly recap card now" ticked.
- **CVE details:** cards that mention CVEs show the CVSS score (NVD), the chance of exploitation in the next 30 days (FIRST EPSS) and an "Exploited" tag for CVEs in CISA's KEV catalog. New CVEs often have no score yet; they are checked again every 6 hours for a week. No keys needed.
- **Owner alerts:** add a repository secret `NTFY_ADMIN_TOPIC` with a second, private ntfy topic and subscribe to it yourself. You get an alert when a run fails (and when it recovers), when AI summaries fail (for example, out of credit), or when a feed has been down for about 3 hours. Readers subscribed to `NTFY_TOPIC` never see these.
- **Day dividers:** the app groups cards by day with a divider screen between days, and has Today and Yesterday filters.

## External 30-minute timer

GitHub often delays scheduled runs by hours on quiet repos. For reliable updates, have a free service such as cron-job.org send `POST https://api.github.com/repos/<owner>/<repo>/actions/workflows/update.yml/dispatches` every 30 minutes, with the body `{"ref":"main","inputs":{"source":"timer"}}`, the headers `Authorization: Bearer <token>`, `Accept: application/vnd.github+json` and `X-GitHub-Api-Version: 2022-11-28`, and a fine-grained token limited to this repo with Actions read and write. Renew the token before it expires.

## Switching AI providers

Add a repository variable (Settings > Secrets and variables > Actions > Variables) named `AI_PROVIDER`:

- `gemini`: create a free key at aistudio.google.com and add it as the secret `GEMINI_API_KEY`. Google may use free-tier prompts to improve its models; the inputs here are public news.
- `anthropic`: add the secret `ANTHROPIC_API_KEY`, buy $5 of credit, and set a spend limit at console.anthropic.com. The app then shows real month-to-date cost.

Model names live in `MODELS` in `pipeline/config.py`. Providers retire models over time; if a run logs a model-not-found error, update the name there.

## Customizing

Everything lives in `pipeline/config.py`:

- `FEEDS`: add or remove RSS sources.
- `WATCHLIST`: vendors and terms you care about. Matches get a tag, and high-severity matches trigger an alert.
- `EXCLUDE_PATTERNS`: title patterns to drop before they reach the AI.
- `QUIET_HOURS`: alerts during these hours arrive silently.
- `DIGEST_HOUR`: time of the morning summary, or `None` to turn it off.
- `NOTIFY_MAX_PER_RUN`: caps alerts per run; any extras are grouped into one notification.

To change how stories are written or rated, edit `SYSTEM_PROMPT` in `pipeline/brief.py`.

If you use a custom domain for Pages, add a repository variable `APP_URL` with the full address so notification links point to the right place.

## Troubleshooting

- **No cards after the first run:** open the run in the Actions tab and read the "Fetch and summarize news" step. A missing secret, a model error or a feed error will be named there.
- **"rate limited" in the log:** the free tier's limit was reached. Remaining stories are picked up on later runs. If it happens daily, switch to `gemini`.
- **Updates arrive late:** GitHub can delay scheduled runs by 5–15 minutes when it's busy.
- **Alerts not arriving:** check that the topic in the ntfy app exactly matches the `NTFY_TOPIC` secret.
- **App shows old cards after an update:** close the app fully and reopen it.

## How it works

- `pipeline/brief.py`: fetches feeds, filters, summarizes in batches of 8 stories per AI call, merges duplicates, sends alerts, and writes `docs/cards.json`.
- `pipeline/state.json`: stories already processed, plus monthly AI usage.
- `docs/`: the web app (`index.html`), offline support (`sw.js`) and install files.
- `.github/workflows/update.yml`: runs the pipeline, commits new cards and deploys the app.
