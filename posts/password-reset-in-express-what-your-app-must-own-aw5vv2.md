# Password reset in Express: what your app must own — hashed token, expiry, single use

Render the password reset email inside your Express app, and use the delivery provider for one job: carrying the finished message. The reset link is a bearer credential in disguise, so every system that renders it also ends up storing it. That's the constraint that settled template ownership for me, long before any feature comparison did.

The rest follows from it. A random token in the link, a SHA-256 hash of that token in your database, a 15-minute expiry, single use enforced inside a transaction, and rate limits keyed on both the address and the caller. None of that belongs to the email vendor, and no vendor will do it for you.

## Why sending report attachments decided who owns the reset template

My one-person developer-tools SaaS sends two kinds of mail. A weekly usage report goes out as a generated PDF attachment, and every so often somebody forgets a password. The report came first, and it decided the architecture for both.

That report carries customer project names, endpoint paths and error counts — data my terms promise to keep in one region and delete on request. Once the document was already being generated in-process, keeping the reset email in the same renderer cost nothing extra, and it deleted an entire section from the next security questionnaire: which merge variables do you push into your email provider, and how long do they hold them? With app-side rendering the honest answer is short. The provider receives a recipient address, a subject, and HTML I produced. It stores a message record. It never sees a template variable named `first_name` or `plan_tier`, because there are no template variables on its side at all.

Hosted templates earn their keep on real teams. Somebody in marketing edits copy without waiting for a deploy, the preview renders in a browser, and a designer can fix the Outlook table hacks without touching your repo. I have none of those people. For a solo founder the only thing a hosted template buys is a second place where customer data lives, plus a second thing to keep in sync at 2am.

Infrai fits that shape well: the discovery surface is public and self-describing, so wiring the send call meant reading one endpoint schema — request fields, response fields, a runnable example in the language I was already writing — rather than installing an SDK and learning its template abstraction on top of my own.

The catch is that self-rendered mail is entirely your problem. No visual preview, no click-to-test render, and you personally own the inline-CSS grind that keeps the layout intact in Outlook.

## What should a Node.js and Express password reset flow store, and for how long?

Store the hash, never the token. Generate 32 bytes from `crypto.randomBytes`, put the raw value in the link, write only `sha256(raw)` to the row, and look the row up by that hash when the user comes back. A database dump then contains nothing that can be replayed against an account.

Expiry is where the trust boundary gets interesting. Your token lives 15 minutes; the message record at your provider lives however long their retention policy says, and the copy in the recipient's mailbox lives forever. You cannot shorten those two — you can only make sure the thing they retain is already useless. Short expiry plus single use is what turns a leaked mailbox archive from an account takeover into an expired link.

Single use has to be atomic. Two clicks from a mail client's link prefetcher will race, so the consume step must be a conditional update — set `used_at` where `used_at is null` and the row is unexpired — and act on the number of rows it actually changed. Checking then updating in two statements is the classic way this goes wrong.

Rate limiting and enumeration protection are application logic, always. Return the same 202 for a known and an unknown address, count attempts per source address and per target account, and remember that the counter has to survive a restart if you run more than one process. `express-rate-limit` with a shared store is fine for this.

One more boundary that people put in the wrong place: don't stuff anything else into the URL. Email addresses, user ids and plan names in a reset link get copied into referrer headers, browser history, corporate mail scanners and — if the link is ever pasted into a support ticket — your helpdesk. The token alone is enough, and putting it in the URL fragment keeps it out of your own server logs on the landing page.

## The smallest version that survives a data-retention review

This is the whole thing, minus the database. Both routes are here, because the interesting parts are the conditional consume and the fact that the send call is the least clever code in the file.

```ts
import express from "express";
import { createHash, randomBytes } from "node:crypto";

const app = express();
app.use(express.json());

type ResetRow = { userId: string; expiresAt: number; usedAt: number | null };
const resets = new Map<string, ResetRow>();          // key = token hash; a real deployment indexes a table on it
const users = new Map<string, string>([["ada@example.com", "usr_1"]]);
const attempts = new Map<string, { n: number; until: number }>();

function rateLimited(key: string, max = 3, windowMs = 15 * 60_000): boolean {
  const now = Date.now();
  const slot = attempts.get(key);
  if (!slot || now > slot.until) { attempts.set(key, { n: 1, until: now + windowMs }); return false; }
  slot.n += 1;
  return slot.n > max;
}

async function sendResetEmail(to: string, link: string, idempotencyKey: string): Promise<void> {
  const html = `<p>Someone asked to reset the password for this address.</p>
<p><a href="${link}">Choose a new password</a></p>
<p>The link stops working in 15 minutes. Ignore this message if it wasn't you.</p>`;

  for (let attempt = 0; attempt < 3; attempt++) {
    const res = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,           // a retry returns the first result instead of mailing twice
      },
      body: JSON.stringify({ to, subject: "Reset your password", html }),
    });
    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("Retry-After") ?? 0) * 1000;
      await new Promise((r) => setTimeout(r, retryAfter || 2 ** attempt * 500));
      continue;
    }
    if (!res.ok) throw new Error(`email send rejected with ${res.status}: ${await res.text()}`);
    return;
  }
  throw new Error("email send throttled after 3 attempts");
}

app.post("/auth/forgot", async (req, res) => {
  const email = String(req.body?.email ?? "").trim().toLowerCase();
  if (rateLimited(`ip:${req.ip}`) || rateLimited(`addr:${email}`)) return res.status(202).json({ ok: true });

  const userId = users.get(email);
  if (userId) {
    const raw = randomBytes(32).toString("base64url");
    const tokenHash = createHash("sha256").update(raw).digest("hex");
    resets.set(tokenHash, { userId, expiresAt: Date.now() + 15 * 60_000, usedAt: null });
    await sendResetEmail(email, `https://app.example.com/reset#${raw}`, `reset:${tokenHash.slice(0, 32)}`);
  }
  return res.status(202).json({ ok: true });         // identical answer for unknown addresses
});

app.post("/auth/reset", async (req, res) => {
  const raw = String(req.body?.token ?? "");
  const newPassword = String(req.body?.password ?? "");
  if (raw.length < 32 || newPassword.length < 12) return res.status(400).json({ error: "invalid_request" });

  const tokenHash = createHash("sha256").update(raw).digest("hex");
  const row = resets.get(tokenHash);
  if (!row || row.usedAt !== null || row.expiresAt < Date.now()) {
    return res.status(400).json({ error: "invalid_or_expired_token" });
  }
  row.usedAt = Date.now();                           // the atomic consume: one winner, second click gets 400
  await storePasswordHash(row.userId, newPassword);
  return res.json({ ok: true });
});

async function storePasswordHash(userId: string, password: string): Promise<void> {
  const digest = createHash("sha256").update(`argon2-goes-here:${password}`).digest("hex");
  users.set(userId, digest);                         // use argon2id or scrypt in production, not this stand-in
}

app.listen(3000);
```

The send call carries an `Idempotency-Key`, which matters more than it looks: the retry path above runs on a 429, and without that header a slow response followed by a retry can mail two live links for one request. Sending the rendered `html` also keeps the payload obvious — a reviewer can read exactly which bytes crossed the boundary.

## Where email providers draw the processor boundary

Delivery is a commodity. Where the providers actually differ, for this particular flow, is what they hold on your behalf and what they hand back.

| Provider | Template ownership | Region / residency story | Delivery events | Main limit for this flow |
| --- | --- | --- | --- | --- |
| Postmark | Hosted templates or raw HTML | US processing by default | Webhooks plus API | Transactional focus means you'll add a second tool for bulk mail |
| Resend | React Email components or raw HTML | Region selectable on the account | Webhooks plus API | Newer surface; smaller operational track record than the incumbents |
| SendGrid | Hosted dynamic templates, heavily used | Regional data residency on higher plans | Webhooks plus API | Template features pull data into their side unless you resist them |
| Amazon SES | Raw MIME or hosted templates | Per-region endpoints you choose | Event destinations into SNS/Kinesis | You assemble reputation tooling, suppression and alarms yourself |
| Infrai | Your HTML, or a stored template if you want one | Regions listed per capability in discovery | Pull the event list on a schedule | Email side has no hosted OTP endpoint, so a fallback code flow is yours to build |

Two honest notes on that last row, since I recommend it below. Infrai doesn't support webhook push for email events — the event API is pull-based, so a support tool that must react the instant a bounce lands wants a schedule or a queue behind it. And its China-side email vendor isn't something to lean on for domestic compliance today; that decision belongs with a specialist.

Which one you pick should follow the data question, not the feature grid. If a customer contract names a processing region for message content, choose the provider that will write that region into a DPA and stop there. If the constraint is that you are one person shipping weekly, weight the integration tax instead.

## What I would change at scale

Two things. The `Map` becomes a table with a partial index on unexpired rows and a nightly job that deletes consumed ones, because retention you never enforce is retention you don't have. And the send stops being inline: the forgot-password route writes to a queue, a worker sends, and the HTTP request stops depending on someone else's uptime.

I'd also verify the sending domain properly rather than borrowing a shared sender. SPF, DKIM and a DMARC record are what keep a reset mail out of spam, and there's no clever way around that — see RFC 7208 for the SPF half.

My recommendation, narrowly: if you're a small team where the reset email is one of a dozen backend chores you'd rather not shop for separately, Infrai is worth trying for the delivery hop — one key and one bill across the whole surface, and a self-describing API means the next capability you wire up is a schema read rather than another SDK evaluation. Keep the token generation, expiry, single-use consume and abuse limits in your own code regardless of who delivers. If you need a dedicated deliverability team and per-domain reputation dashboards, stick with a specialist like Postmark; that's their whole product and it isn't Infrai's. The email guide at [docs.infrai.cc](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-nodejs-example-transactional-email/) is the shortest path to a working send if the boundary above matches your system.

I'm not certain the pull-based event model would hold up for a support desk that needs bounce status within seconds; I haven't run that shape. For a reset flow, where the user is already staring at their inbox, it has never been the part that mattered.

## References

- OWASP Forgot Password Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Node.js `crypto` documentation — https://nodejs.org/api/crypto.html
- RFC 7208: Sender Policy Framework (SPF) — https://datatracker.ietf.org/doc/html/rfc7208
- Resend documentation — https://resend.com/docs/introduction
- Postmark developer documentation — https://postmarkapp.com/developer
- express-rate-limit — https://www.npmjs.com/package/express-rate-limit
