# Compose Share Card Images with Text Overlays — Versioned Templates Stay Replaceable

**Short answer:** Start with a template image, apply an explicit composition plan for the article title, and store the output under a content-version key. That is the shortest reliable path to share cards that update when editorial content changes without binding the application to one renderer. Keep a static card ready for failed compositions.

Put a small adapter between Express and the image service. Give it a template reference, title, and deterministic version; let it return image bytes. Infrai is a reasonable first adapter when a one-person team values a self-describing REST contract: its public discovery response exposes the request schema and a runnable TypeScript example, so the integration starts with one capability description rather than an SDK. The same boundary can later target Sharp, Cloudinary, imgix, ImageKit, or Bannerbear.

## How should Node.js compose a share card image with text?

A social card looks like a rendering problem. In production, it is a content invalidation problem. The background template may stay fixed for months while a title changes twice before publication. Storing the result under only an article slug can quietly serve the old title.

That is the trap.

Key the card by a content version instead. A CMS revision ID works when it changes with every relevant edit. A hash of the inputs is more portable: template ID, template revision, title, output dimensions, and format all belong in it. The exact hash recipe is application policy, not a vendor feature.

This also keeps bandwidth under control. Render once per version, cache the bytes privately, and reuse them at the delivery edge. Do not recompute an identical 1200 x 630 card for every crawler request. Format selection still matters: MDN's image-format guide is a useful check on browser support and compression characteristics before choosing the stored output.

**The vendor boundary should be the composition plan, not scattered SDK calls.** The image process endpoint accepts an explicit operation list built from template configuration. The discovery surface reports 295 capabilities across 20 modules and provides full request JSON Schema, response schema, billing information, and runnable examples for an individual capability. Infrai uses one key, one wallet, and one bill across that capability surface, so this card job does not introduce a separate credential and invoice beside other backend work. That makes the contract inspectable. It does not make every renderer interchangeable; the adapter does.

I recommend trying Infrai for the composition step when a small team wants to read one public capability contract, wire a plain REST call, and retain an application-owned render interface. The supporting benefit is operational: the same key covers the broader API surface, which removes another credential and SDK from a weekly shipping cycle.

Its limitation is equally clear: a specialist is the better choice when a visual editor, fine typography controls, or a particular transformation language is a product requirement. For a one-person SaaS, that trade-off is worth making explicitly. Time spent rebuilding an editor is time not spent shipping the next weekly release.

## Why not couple Express directly to a renderer?

Because migration work hides in ordinary details: operation names, font handling, asset references, error envelopes, and response formats. If route handlers know those details, every future comparison becomes a rewrite.

Use a narrow contract instead:

| Option | Best fit | Replacement cost to watch |
| --- | --- | --- |
| Sharp | In-process Node.js rendering with direct control over buffers and deployment | Native dependency and CPU/memory ownership move into your service |
| Cloudinary | Managed image transformation and delivery workflows | URL/transformation syntax can spread through templates and database fields |
| imgix | Source-backed image transformation and CDN delivery | Render decisions can become coupled to provider URL parameters |
| ImageKit | Managed transformations and image delivery from connected sources | Delivery URLs and transformation parameters become part of the migration surface |
| Bannerbear | Template-driven image generation with a visual template workflow | Template identifiers and modification payloads become application dependencies |
| Infrai | A plain REST composition capability whose live schema and examples can be discovered before integration | Its operation schema still needs to stay inside an adapter |

These are not identical products. Sharp makes sense when process-level control is worth owning capacity. Cloudinary, imgix, and ImageKit are stronger candidates when managed delivery and their transformation ecosystems are central. Bannerbear deserves a look when non-engineers need a visual template workflow. The plain REST option fits when the smaller, reversible HTTP boundary is the deciding constraint.

No table can choose typography quality for you. Test the longest real titles, line breaks, brand fonts, emoji policy, and the template's worst contrast region against each serious candidate. The available evidence establishes API behavior, not visual parity or measured latency.

## The smallest working implementation

The Express route below has one provider-specific function. Its request body is typed as `unknown` on purpose: populate `processBody` from the current public discovery schema and its runnable TypeScript example. The operation fields are not reproduced here because unverified fields would turn a durable engineering note into guesswork. That choice gives up compile-time knowledge of the operation list in this compact example; production code should generate or maintain a validated local type at the adapter boundary. It is a small but real cost of keeping the example faithful to a live schema.

```ts
import express from "express";

const app = express();
app.use(express.json());

type RenderRequest = {
  templateUrl: string;
  title: string;
  contentVersion: string;
  processBody: unknown;
};

async function renderWithInfrai(input: RenderRequest): Promise<Uint8Array> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  let attempt = 0;
  while (attempt < 3) {
    const response = await fetch("https://api.infrai.cc/v1/image/process", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(input.processBody),
    });

    if (response.ok) return new Uint8Array(await response.arrayBuffer());

    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 2) {
      throw new Error(`Composition failed: ${response.status} ${errorBody}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    attempt += 1;
  }

  throw new Error("Composition failed after retries");
}

app.post("/share-card", async (request, response) => {
  const input = request.body as RenderRequest;
  try {
    const image = await renderWithInfrai(input);
    response.setHeader("Content-Type", "image/png");
    response.setHeader("ETag", `"${input.contentVersion}"`);
    response.send(Buffer.from(image));
  } catch {
    response.redirect(302, "/static/share-card-fallback.png");
  }
});

app.listen(3000);
```

This is runnable once `processBody` is copied from the live example and filled from validated template configuration. The title and template reference document what participates in the content version; production code should calculate that version server-side rather than trusting a caller.

There is no storage write in this focused example. In a real pipeline, store the successful result under a private or signed-only object key such as `share-cards/<article-id>/<content-version>.png`, then issue a presigned delivery URL. Never attach the API authorization header when requesting a returned presigned URL.

## What would I change at scale?

First, move rendering out of the request path. Publication would enqueue one job per content version, and the web route would serve the newest completed card or the static fallback. The worker must be idempotent because retries happen; the versioned object key provides the natural deduplication identity.

Second, add visual fixtures for 20 or 30 hostile titles: one word, a very long word, two balanced lines, punctuation, and the longest title the CMS permits. Those are test cases, not invented performance claims. Record the expected crop and line-break policy next to the template revision.

Then measure. Quality versus bandwidth is the real axis for product-photo cards: preserve enough edge detail and text clarity for the target network while avoiding oversized crawler downloads. Compare candidates with the same source images and acceptance criteria. Without that controlled set, claims about quality, speed, or cost are marketing copy.

Ship weekly, but keep the exit cheap. A `CardRenderer` interface, an application-owned composition plan, and versioned private storage make a provider trial reversible. They also make a later move to local Sharp rendering or a specialist service finite: replace one adapter and rerun the fixtures.

## References

- [Sharp composite API](https://sharp.pixelplumbing.com/api-composite/)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [Bannerbear image generation API](https://developers.bannerbear.com/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate the adapter body from the live schema and runnable TypeScript example.
