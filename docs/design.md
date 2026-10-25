# Media: Design

> **In short:** media holds the images a site's documents use. A group owns each image, and uploading asks Authorisation. Every image is made web-ready on upload: resized, compressed as tightly as it will go, and stripped of location and other metadata. A group may also keep a cleaned full-resolution original, which is private. Alt text is required, and who uploaded it is always recorded. The records live in media's own PostgreSQL schema, and the files in a storage bucket the host passes in. Each group has a storage allowance, set by an entitlement. Agreed with the owner on 25 October 2026, apart from the questions at the end; nothing is built yet.

## What it is for

content-editor's design ([content-editor](https://github.com/nuxt4-layers/content-editor/blob/4bdcbe93120cb448ef8705d3b5fc46dca6597d64/docs/design.md)) needs images in documents. Publisher refers to an image as `asset:<id>` and asks its asset port for the address ([Publisher contract](https://github.com/nuxt4-layers/publisher/blob/4410cab7b84c1b0395de2bdf657e3bcc3d476b87/docs/contracts.md) §6.2, §11). media answers that port and offers content-editor a picker. A later gallery or photo of the week draws on the same images: media stores them, and the gallery (most simply a Publisher document type listing `asset:` ids) chooses, orders and captions them.

## Decisions

| Decision | Means |
|---|---|
| Records here, files in a bucket | Each item's record (owning group, uploader, alt text, credit, kind, sizes, status) is in media's own schema ([ADR-0002](https://github.com/nuxt4-layers/platform-architecture/blob/249cd48af72e0a3fc232d40c8aa05e8da3b32069/docs/decisions/ADR-0002-composition-supplied-persistence-and-capability-owned-schemas.md)). The files are in a bucket the host passes in, through a small storage port (`put`, `delete`, `url`, and a signed upload address), so the bucket can be changed without changing media. Files are kept under the tenant's and group's identifiers, never under names people chose |
| Accepted uploads | JPEG, PNG, WebP, AVIF and HEIC (an iPhone's default), up to 10 MB each, or the group's larger limit (below). The file's content is checked, not its name. Images over 50 megapixels are refused before decoding, so a small file that expands enormously cannot exhaust the server. Anything else is `validation-failed` |
| Uploads go straight to the bucket | Vercel refuses requests over about 4.5 MB, so the browser uploads to a short-lived, single-use address in the bucket, and media then processes the file. An upload that is never processed is removed within a day |
| Web-ready on upload | Each image is decoded and re-encoded once, as WebP at 480, 960 and 1920 pixels wide (never wider than the original), compressed as tightly as it will go without visible loss. Re-encoding also discards anything hidden in the file. Processing has time and memory limits; one that fails or runs over leaves nothing behind and says so |
| Metadata removed | Location, camera and every other metadata is always removed, from web versions and originals alike |
| Originals off by default | A group whose entitlement allows it keeps a full-resolution original as well. It is cleaned too: a JPEG's metadata is removed without re-encoding, so the picture is untouched; other formats are re-encoded at full size and the highest quality. Originals are never public: they are served through the site to holders of `media.originals:download` |
| Alt text required | An image cannot be inserted until it has alt text, or is marked decorative on purpose (alt left empty deliberately) |
| Credit and rights | The uploader confirms they have the right to use the image. A credit line is optional and is shown wherever the image is |
| A group owns each image | As with content, an image belongs to a group. Uploading needs `media.items:upload` in that group; using an image in a document needs the right to edit that group's content. All through the shared `decide` shape ([ADR-0012](https://github.com/nuxt4-layers/platform-architecture/blob/249cd48af72e0a3fc232d40c8aa05e8da3b32069/docs/decisions/ADR-0012-trivial-composition.md) §2) |
| The uploader is always recorded | Each item keeps its uploader's identity identifier, and screens show the uploader's name through Profile. Events and logs carry identifiers only |
| Allowances by entitlement | media counts each group's storage. Its allowance (total size, keeping originals, the larger upload limit) comes from an entitlement port, defaulting to a free allowance the host sets (100 MB proposed). A paid subscription or an exception, such as always free for islanders, raises it. Until those capabilities exist, the platform's administrators set allowances by hand. Media never handles payment |
| Over the allowance, refuse | Uploads beyond the allowance are refused with the use and the limit shown. Nothing is ever deleted for being over: a lowered allowance, such as a lapsed subscription, stops new uploads only |
| No deletion while in use | Deleting is refused while a document links to the image, and the refusal lists those documents, as with Publisher's `acknowledgedLinks`. Archiving hides an image from the picker without breaking documents that use it |
| Unused images tidied | An image no document has used within 90 days of upload is listed to its group's administrators for deletion; nothing is removed without them |
| Erasure leaves a blank | When an image must go (the uploader's or a pictured person's erasure, or a takedown), its files and original are deleted and it answers a blank image, at a new address so that cached copies are not served, with its alt text set to a neutral "Image removed". The `asset:` id still resolves, so documents never break. On the uploader's erasure, their identifier is removed from the record |
| Reporting an image | Every public image can be reported. A report goes to the group's administrators and the platform's operators, who keep the image or take it down; reports of the most serious kinds (the platform's list) hide it at once while they decide. Reports and decisions are events for the audit capability. This is the first part of the Online Safety Act duties; slice 3 adds the rest |
| Backups in step | Records and files are backed up together, so a restore never leaves records without files or the reverse. Each erasure is recorded, so a restore applies it again |

## Permissions

| Permission | Risk | Allows |
|---|---|---|
| `media.items:view` | low | Seeing a group's images in the picker and library |
| `media.items:upload` | low | Uploading to the group, and editing alt text and credit |
| `media.items:archive` | medium | Archiving and restoring |
| `media.items:delete` | medium | Deleting an unused image |
| `media.originals:download` | medium | Downloading a kept original |
| `media.reports:decide` | medium | Keeping or taking down a reported image |
| `media.allowances:manage` | high | Setting a group's allowance by hand (platform operators only) |

## Screens

A presentation part ([ADR-0009](https://github.com/nuxt4-layers/platform-architecture/blob/249cd48af72e0a3fc232d40c8aa05e8da3b32069/docs/decisions/ADR-0009-headless-capability-layers.md)), for people who edit: a picker component that content-editor uses, a library page per group at `/media/:groupId` showing its use against its allowance, and a reports page at `/media/:groupId/reports`. The report button on public images is a small component the host places. All are styled only with Theme Manager's vocabulary, worded from an en-GB catalogue, accessible in light and dark mode, and can be switched off with `media: { presentation: false }`.

## Where the files go for Westray-Papa-Westray

The owner chose Vercel's storage, Vercel Blob, as near free as possible. Its Hobby plan includes 1 GB of storage and 10 GB of transfer a month at no charge, and stops serving (rather than charging) when a limit is exceeded ([usage and pricing](https://vercel.com/docs/vercel-blob/usage-and-pricing)). London (`lhr1`) has its own Blob rates, but Vercel lists regional pricing for Pro plans only ([London pricing](https://vercel.com/docs/pricing/regional-pricing/lhr1)).

- **To confirm before any data is stored:** whether a Hobby Blob store can be created in London. The site's data stays in the UK. If it cannot, the choice is between the Pro plan and a UK S3-compatible bucket; the storage port makes either a configuration change.
- **Limits stop every image, not only new ones.** On Hobby, reaching the storage or transfer limit stops all images being served, and transfer is the likelier limit. media reports use against the bucket's limits as a scheduled task and warns the operators at 70% and 90%. Before the site depends on images, the owner chooses between a paid plan and refusing uploads at 90%.
- **The site pays for all storage,** free groups included, so the free allowance is the owner's own cost per group.

## Not in version 1

Images in members-only content (served through the site with an access check, as originals are), video, documents such as PDFs, AVIF output alongside WebP, blurred placeholders, captions and long descriptions, cropping and focal points, albums, download statistics, and moving between storage providers.

## Open for the owner

Recommended answers, to confirm before the core is built:

1. **Photos of children:** the uploader confirms that anyone identifiable under 18 is shown with a parent's or guardian's consent; without it the upload is refused.
2. **Reporting now:** the report button and takedown come with version 1, as above, rather than waiting for slice 3.
3. **Credit:** optional, as above, but required for any image the uploader did not take.
4. **Free allowance:** 100 MB per group, and 25 MB as the larger upload limit for groups keeping originals.

## Order of work

1. Confirm the Blob store's region, and settle the questions above.
2. media's core: contract, storage port, direct uploads, processing, records, allowances, Publisher's asset resolver, fakes and tests.
3. Its presentation part: the picker, the library and reports.
4. `examples/minimal-host`, `CONTRACT.md` and a release; then content-editor builds on it.
