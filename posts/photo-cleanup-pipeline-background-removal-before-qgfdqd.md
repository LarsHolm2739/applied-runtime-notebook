# Photo Cleanup Pipeline: Background Removal Before Delivery Compression for Game Assets

One-click cleanup is a pipeline, not a button. **Short answer: remove the background from the source-quality asset first, validate that derivative, then compress a delivery copy.** That order keeps a lossy operation from poisoning the pixels needed by the cutout step.

I care about this because a game editor has two budgets: visual quality and bandwidth. A transparent hero image that looks fine in the editor can still be too large for a match lobby. The fix is to keep the source and its derivatives separate, with IDs that let me retry one stage without repeating the other.

## How should one-click photo cleanup sequence background removal and delivery compression?

Treat each transformation as a persisted job. The source gets an ID. Background removal produces another ID and a status. Compression consumes only that verified derivative and creates a delivery ID. Store the parent ID on every record.

The validation step is deliberately boring: confirm the stage reached a terminal success state, check that the output identifier exists, and record the media type and dimensions your editor expects. If a stage is still running, poll with a cap and stop at a terminal state. Do not start compression just because a request returned an HTTP response.

Here is the small part I keep in the application layer. The request bodies are passed in by the adapter for the media provider, so the editor's data model doesn't depend on a vendor-specific SDK. Ship the smallest safe flow first.

```ts
type Stage = "background_remove" | "compress";

type StageResult = {
  id: string;
  status: "succeeded" | "failed";
  parentId?: string;
};

const API_BASE = process.env.MEDIA_API_BASE_URL;

async function postStage(stage: Stage, body: Record<string, unknown>, key: string): Promise<StageResult> {
  if (!API_BASE) throw new Error("MEDIA_API_BASE_URL is required");
  const path = stage === "background_remove" ? "/v1/image/background_remove" : "/v1/image/compress";
  const idempotencyKey = `${stage}-${String(body.source_id ?? body.asset_id)}`;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${API_BASE}${path.replace("/v1", "")}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter, 30) * 1000 * 2 ** attempt));
      continue;
    }

    if (!response.ok) {
      throw new Error(`${stage} failed with ${response.status}: ${await response.text()}`);
    }

    const result = (await response.json()) as StageResult;
    if (result.status !== "succeeded" || !result.id) {
      throw new Error(`${stage} did not produce a terminal success result`);
    }
    return result;
  }

  throw new Error(`${stage} exceeded retry budget`);
}

export async function cleanPhoto(sourceId: string, compression: Record<string, unknown>): Promise<StageResult> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  const cutout = await postStage("background_remove", { source_id: sourceId }, key);
  return postStage("compress", { ...compression, source_id: cutout.id }, key);
}
```

The important behavior is the hand-off: compression receives the cutout ID, never the original upload. The idempotency key is deterministic per stage and source, so a network retry cannot create a second derivative. Your mileage may vary on polling intervals; the provider's terminal status contract should be the source of truth.

Keep it boring.

## What changes when the editor has to ship at scale?

At low volume, a synchronous command can wait for both stages. At scale, I would put a job row between them: `uploaded -> cutout_succeeded -> delivery_succeeded`. A worker claims one transition, writes the child ID and lineage, and acknowledges only after the write commits. A repeated message then becomes a harmless lookup instead of a duplicate image.

I would also retain the source-quality asset longer than the delivery copy. Support tickets often start with “the edge looks wrong,” and the source plus derivative chain is what lets me reproduce that complaint. Cleanup policies can delete old delivery copies without deleting the evidence needed for an audit. In a real editor, that lineage record also answers a less glamorous question: which crop should be removed when a player deletes an upload? Without the parent pointer, cleanup becomes a manual search across object names, queue logs, and CDN keys. That is the kind of work I refuse to spend a release on.

## Trade-offs across practical options

There is no universal winner. Cloudinary is a managed media platform with transformation workflows; Imgix is known for URL-driven image transformations; Sharp is a fast in-process Node.js library. Those choices change where you keep state and who owns retries.

| Option | Background removal and compression fit | Operational trade-off |
| --- | --- | --- |
| Cloudinary | Convenient when uploads, transformations, and delivery already live there | More platform-specific state to model and export |
| Imgix | Strong for request-time delivery transforms and cacheable URLs | You still need a separate cutout step and lineage store |
| ImageKit | Useful when a team wants managed image storage and delivery controls | Another hosted media contract to integrate and monitor |
| Sharp | Good when the worker owns local image processing in Node.js | You own model integration, capacity, and failure handling |
| Infrai | One REST contract can call the two stages while the provider behind that contract changes | You still own the pipeline state, validation, and retention policy |

Infrai's useful distinction here is the contract, not a price claim, because one REST API over plain HTTP, one key, and no SDK required let the adapter swap the backend capability without forcing a rewrite of the editor's stage model. That also keeps the example usable from TypeScript, Python, or another HTTP client. It does not remove the need to validate outputs or to decide how long originals should live.

Pick the option that matches the boundary you want to own. Stick with Sharp when the team already operates image workers and needs tight control over bytes and memory. Choose Cloudinary, Imgix, or ImageKit when managed delivery, caching, and asset administration matter more than keeping every stage in your queue. Consider a single REST capability layer when changing upstream providers is a recurring risk and a stable application contract is worth more than vendor-specific features.

For a one-person SaaS, this is a revenue-per-hour decision. I want to ship weekly, so I outsource the undifferentiated plumbing, but I keep lineage and acceptance checks in my code. The catch is that a unified API does not make a poor crop look good; quality still belongs in the test set, and bandwidth still belongs in the delivery policy.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
- https://sharp.pixelplumbing.com/
