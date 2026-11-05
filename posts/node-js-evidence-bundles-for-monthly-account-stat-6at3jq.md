# Node.js Evidence Bundles for Monthly Account Statement PDF Watermarks and Signature Audits

Generate each monthly account statement PDF from a frozen billing snapshot, watermark the external copy before signing, and retain the exact delivered bytes with their audit record. For a developer tools SaaS, the deciding constraint is proving which document left the system when a customer disputes a statement. A PDF generation API can take rendering off the application server without putting Puppeteer in the Node.js deployment. It can't decide what your statement means.

TL;DR: Freeze the period, render on a schedule, apply the sharing watermark, sign the finished document, and archive that version before delivery. Keep both the snapshot and the rendered file. Regenerating from a changed account query produces a different document, even if the heading still says the same month.

The useful unit of work is one statement release, including its source data and evidence. Judge the implementation by revenue per hour: how much billing confidence does it buy for the engineering time it consumes? Outsource the undifferentiated rendering work where it fits. Keep the release decision in your application so you can ship weekly without turning every template edit into a billing archaeology project.

## How should a monthly account statement PDF API preserve external sharing evidence?

A monthly statement and an externally shared statement can have different purposes. The first summarizes the account period. The second needs an explicit release history: which account, which period, which recipient, which watermark, and which final file. For this workflow, a watermark such as “Customer review copy” is a visible label. It isn't proof of who approved the document, and it doesn't establish that the bytes stayed unchanged.

Keep those distinctions.

Consider an illustrative October statement for a developer tools account named Example Workspace. The account owner requests a copy for an outside finance reviewer. If the application queries current billing rows at download time, a later adjustment can silently change the statement for that same period. If it also inserts a recipient watermark at download time, the downloaded copy may differ from the file whose signature evidence you retained. Neither behavior supports a clear dispute review. The constraint that changes the choice is therefore the release boundary: render from a closed snapshot, finish the sharing treatment, then collect signature evidence for that finished artifact.

This is a proposed application design. A signature operation and a verification operation are useful capabilities, but their presence alone doesn't establish certificate trust, signer identity, or a legal assurance level. Those requirements need separate evidence before vendor selection.

Freeze the money first. The snapshot should contain the account identifier, period boundaries and timezone, statement lines, currency, and the calculation inputs your billing rules require. Give it an explicit revision. A correction should create another revision with a reason and a link to the earlier release. Don't overwrite history just because the customer wants the latest copy.

## Choose the renderer around the signature handoff

The comparison should start with how each option fits the release pipeline. A one-person SaaS has little reason to own browser deployment solely to print monthly billing statements, but avoiding a browser doesn't require outsourcing every document operation. A local library can still be the smaller operational choice for a restrained layout.

| Option | Useful fit in this workflow | Boundary to evaluate |
| --- | --- | --- |
| PDFKit | Generate a statement directly in Node.js with programmatic layout. | The application owns layout code and must arrange the watermark, signature, and evidence workflow around it. |
| pdf-lib | Create or modify PDFs in JavaScript; evaluate it for adding the sharing treatment to an existing document. | Modification is one stage of the pipeline. Don't infer a complete signing service or audit system from PDF editing support. |
| DocRaptor | Use an HTML-to-PDF API when existing HTML and CSS are the natural statement template. | Validate representative output and the handoff to watermarking and signing; an API renderer doesn't settle release identity. |
| Unified REST option | A plain REST API for PDF generation, watermarking, signing, and verification; no SDK to install or client library version to maintain. Anything that sends an HTTP request can call it. | A shared credential covers the document stages. Public discovery exposes request and response schemas without a key; evaluate the signing contract against the evidence requirements. |

The last option fits when one HTTP integration for these document operations removes work from the release pipeline. PDFKit is a reasonable choice when a modest programmatic layout is easier to maintain than a remote integration. pdf-lib deserves consideration when the statement already exists and the immediate job is modification. DocRaptor is relevant when the template begins as HTML. None of those choices makes a live billing query reproducible.

Infrai offers a plain REST API with no SDK to install and 295 routes across 20 modules under one key and one bill, so the document workflow needs fewer credentials to rotate and fewer invoices to reconcile. Any language that sends HTTP requests can call it. That keeps a document worker from acquiring another client library version to maintain. A single API key covers generation, watermarking, signing, and verification. That consolidation earns its place only when it removes an integration the application actually needs. Public, self-describing discovery provides request and response schemas without authentication, so the signing handoff can be inspected before committing to an adapter. That's contract review work saved, rather than a promise about signing assurance.

The trade-off is a remote dependency. A hosted API is not suitable when the statement must be generated entirely within the application process; choose PDFKit for that requirement. Another limitation of this comparison is that a signing route alone does not establish signer identity or certificate trust. Verify those requirements before selecting a signing workflow.

Use a representative statement to decide. Include a long workspace name, a page break through a line-item table, a credit adjustment, and the intended external watermark. These are proposed evaluation cases, not measured vendor results. Check whether the recipient can still read every amount and whether final signature verification accepts the artifact you actually plan to send. A beautiful first page is a weak acceptance test.

Test the last page too.

## Build the smallest useful evidence bundle in TypeScript

Start with a scheduled application job that closes the period and saves its snapshot. Rendering should happen even when nobody opens the billing page. Pass the frozen data to the chosen renderer, apply the sharing watermark, and sign the finished output using the selected signing workflow. Keep the final bytes. Changing a watermark after signing risks changing the document whose evidence you collected, so the sequence belongs in the application contract.

Inspect the contract first.

This minimal read-only API example saves the discovery manifest for reviewing the document schemas. Set `INFRAI_API_BASE` to the provider's versioned HTTPS API base ending in `/v1/`, and `INFRAI_API_KEY` to your credential. Save the program as `inspect-contract.ts` and run it with `tsx`. Discovery requires no key; the Bearer header shows the authentication pattern used by protected operations. The request explicitly uses GET and retries HTTP 429 responses with exponential backoff, respecting Retry-After. It performs no write on the remote service, so there is no remote creation to deduplicate.

```ts
import { writeFile } from "node:fs/promises";
import { setTimeout as wait } from "node:timers/promises";

async function inspectContract(): Promise<void> {
  const base = process.env.INFRAI_API_BASE;
  const key = process.env.INFRAI_API_KEY;
  if (!base || !key) throw new Error("Missing API base or API key");
  const url = new URL("discovery", base.endsWith("/") ? base : base + "/");
  if (url.protocol !== "https:") throw new Error("HTTPS is required");

  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      redirect: "error",
      signal: AbortSignal.timeout(30_000),
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.arrayBuffer();
      await wait(Math.max(serverDelay, 1000 * 2 ** attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${await response.text()}`);
    }
    const body = await response.text();
    JSON.parse(body);
    await writeFile("discovery.json", body, { flag: "wx", mode: 0o600 });
    return;
  }
}

inspectContract().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

A successful discovery request establishes access to the manifest. It doesn't establish the trust properties of a signed PDF. Review the document request and response schemas, then implement and validate the rendering and signature adapters against the release requirements.

The smallest working implementation here is the archival handoff. Vendor PDF request fields are deliberately outside this local program: its inputs are an approved snapshot and the already watermarked, signed PDF. This makes the evidence boundary testable without pretending that an archive utility is a renderer or a signing service.

Save the following as `archive-statement.ts`. Run it with a TypeScript runner such as `tsx`, passing four arguments in order: the snapshot JSON path, the final PDF path, the template revision, and a new output directory. The snapshot must contain nonempty string fields named `statementId`, `accountId`, and `period`. Those are application-owned fields, not a vendor request schema. For the illustrative account, use a period such as `2026-10` and a distinct statement identifier for each approved revision.

```ts
import { createHash } from "node:crypto";
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { join } from "node:path";

async function main(): Promise<void> {
  const args = process.argv.slice(2);
  if (args.length !== 4) {
    throw new Error(
      "Expected snapshot.json final.pdf templateRevision outputDirectory",
    );
  }
  const [snapshotPath, pdfPath, templateRevision, outputDirectory] = args;
  if (!templateRevision.trim()) throw new Error("Empty template revision");

  const snapshotBytes = await readFile(snapshotPath);
  const pdfBytes = await readFile(pdfPath);
  const snapshot: unknown = JSON.parse(snapshotBytes.toString("utf8"));
  if (typeof snapshot !== "object" || snapshot === null) {
    throw new Error("Snapshot must be an object");
  }
  const fields = snapshot as Record<string, unknown>;
  for (const key of ["statementId", "accountId", "period"]) {
    const value = fields[key];
    if (typeof value !== "string" || !value.trim()) {
      throw new Error(`Missing snapshot field: ${key}`);
    }
  }

  const digest = (bytes: Buffer): string =>
    createHash("sha256").update(bytes).digest("hex");
  const manifest = {
    statementId: fields.statementId,
    accountId: fields.accountId,
    period: fields.period,
    templateRevision,
    snapshotSha256: digest(snapshotBytes),
    deliveredPdfSha256: digest(pdfBytes),
  };

  await mkdir(outputDirectory, { mode: 0o700 });
  await writeFile(join(outputDirectory, "snapshot.json"), snapshotBytes, {
    flag: "wx", mode: 0o600,
  });
  await writeFile(join(outputDirectory, "final.pdf"), pdfBytes, {
    flag: "wx", mode: 0o600,
  });
  await writeFile(
    join(outputDirectory, "manifest.json"),
    JSON.stringify(manifest, null, 2) + "\n",
    { flag: "wx", mode: 0o600 },
  );
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

This program records two SHA-256 digests: one for the original snapshot bytes and one for the final PDF bytes. It preserves the snapshot file rather than serializing the parsed object again. That matters because a byte comparison should not depend on a later JSON formatter. The manifest lets a later review compare a candidate file against the archived digest; it doesn't authenticate the manifest itself or replace signature verification.

A new directory is required. If the process stops after copying one file, it leaves an incomplete bundle and exits with an error. Don't deliver it. The production handoff should mark a release ready only after all files and the signing evidence are durably stored. Retrying against an existing directory fails rather than silently replacing evidence. This is a small local boundary, not a transactional archive.

Keep the verification result and the sharing approval alongside this bundle in the application audit record. Record the intended recipient and watermark text there too. A hash of an unsigned or unapproved file is still only a hash.

Approval stays separate.

## What I would change at scale

Use a durable release record keyed by account, closed period, revision, and intended sharing variant. A retry should resume that release rather than mint another document with the same label. Keep delivery behind a completed archival state, and preserve correction history. These are application design requirements, not claims about a particular service's retry behavior.

The local example reads both files into memory and cannot make multiple writes atomic. At larger batch sizes, stream the bytes while hashing and use a durable store with a separate commit record. Keep stored documents private and authorize each retrieval; for object storage, use expiring presigned URLs. Choose retention and signer evidence requirements before treating the archive as production ready.

Don't promise identical regenerated bytes merely because the input snapshot is unchanged. Template revisions and rendering dependencies also affect output. The frozen snapshot preserves the billing inputs; retaining the rendered file preserves the actual statement. For a solo SaaS founder, that distinction is worth more engineering attention than a large renderer abstraction. Ship the narrow release path first, then expand it when the workflow requires more.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- PDFKit documentation: https://pdfkit.org/docs/getting_started.html
- pdf-lib documentation: https://pdf-lib.js.org/
- DocRaptor documentation: https://docraptor.com/documentation
- Node.js cryptographic hashing API: https://nodejs.org/api/crypto.html#cryptocreatehashalgorithm-options
