# Media: Design

> **In short:** media holds the images a site's documents use. A group owns each image. Uploading asks Authorisation. Every image is made web-ready on upload: resized, compressed as tightly as it will go, and stripped of location and other metadata. Alt text is required, and who uploaded it is always recorded. The records live in media's own PostgreSQL schema, and the files in a storage bucket the host passes in. Agreed with the owner on 25 October 2026; nothing is built yet.

## What it is for

content-editor's design ([content-editor](https://github.com/nuxt4-layers/content-editor/blob/4bdcbe93120cb448ef8705d3b5fc46dca6597d64/docs/design.md)) needs images in documents. Publisher refers to an image as `asset:<id>` and asks its asset port for the address ([Publisher contract](https://github.com/nuxt4-layers/publisher/blob/4410cab7b84c1b0395de2bdf657e3bcc3d476b87/docs/contracts.md) §6.2, §11). media answers that port and offers content-editor a picker.

## Decisions

| Decision | Means |
|---|---|
| Records here, files in a bucket | Each item's record (owning group, uploader, alt text, kind, sizes, status) is in media's own schema ([ADR-0002](https://github.com/nuxt4-layers/platform-architecture/blob/249cd48af72e0a3fc232d40c8aa05e8da3b32069/docs/decisions/ADR-0002-composition-supplied-persistence-and-capability-owned-schemas.md)). The files are in a bucket the host passes in, through a small storage port (`put`, `delete`, `url`), so the bucket can be changed without changing media |
| Accepted uploads | JPEG, PNG, WebP and AVIF, up to 10 MB each. The file's content is checked, not its name. Anything else is `validation-failed` |
| Web-ready on upload | Each image is decoded and re-encoded once, on upload, as WebP at 480, 960 and 1920 pixels wide (never wider than the original), compressed as tightly as it will go without visible loss. Re-encoding also discards anything hidden in the file |
| No original kept | Only the web versions are stored, keeping the site within Vercel Blob's free 1 GB. An original is kept only if a later need is agreed |
| Metadata removed | Location, camera and every other metadata is always removed, whatever the uploader chooses |
| Alt text required | An image cannot be inserted until it has alt text, or is marked decorative on purpose (alt left empty deliberately) |
| A group owns each image | As with content, an image belongs to a group. Uploading needs `media.items:upload` in that group; using an image in a document needs the right to edit that group's content. All through the shared `decide` shape ([ADR-0012](https://github.com/nuxt4-layers/platform-architecture/blob/249cd48af72e0a3fc232d40c8aa05e8da3b32069/docs/decisions/ADR-0012-trivial-composition.md) §2) |
| The uploader is always recorded | Each item keeps its uploader's identity identifier, and screens show the uploader's name through Profile. Events and logs carry identifiers only |
| No deletion while in use | Deleting is refused while a document links to the image, and the refusal lists those documents, as with Publisher's `acknowledgedLinks`. Archiving hides an image from the picker without breaking documents that use it |
| Erasure leaves a blank | When a person's data must be erased (the right to erasure) and their images go with it, each image's files are replaced with a blank image and its alt text with a neutral "Image removed". The `asset:` id still resolves, so documents never break, and the uploader's identifier is removed from the record |

## Permissions

| Permission | Risk | Allows |
|---|---|---|
| `media.items:view` | low | Seeing a group's images in the picker and library |
| `media.items:upload` | low | Uploading to the group, and editing alt text |
| `media.items:archive` | medium | Archiving and restoring |
| `media.items:delete` | medium | Deleting an unused image |

## Screens

A presentation part ([ADR-0009](https://github.com/nuxt4-layers/platform-architecture/blob/249cd48af72e0a3fc232d40c8aa05e8da3b32069/docs/decisions/ADR-0009-headless-capability-layers.md)), for people who edit: a picker component that content-editor uses, and a library page per group at `/media/:groupId`. Both are styled only with Theme Manager's vocabulary, worded from an en-GB catalogue, accessible in light and dark mode, and can be switched off with `media: { presentation: false }`.

## Where the files go for Westray-Papa-Westray

The owner chose Vercel's storage, Vercel Blob, as near free as possible. Its Hobby plan includes 1 GB of storage and 10 GB of transfer a month at no charge, and stops serving (rather than charging) when a limit is exceeded ([usage and pricing](https://vercel.com/docs/vercel-blob/usage-and-pricing)). London (`lhr1`) has its own Blob rates, but Vercel lists regional pricing for Pro plans only ([London pricing](https://vercel.com/docs/pricing/regional-pricing/lhr1)).

**To confirm before any data is stored:** whether a Hobby Blob store can be created in London. The site's data stays in the UK. If it cannot, the choice is between the Pro plan and a UK S3-compatible bucket; the storage port makes either a configuration change.

## Not in version 1

Images in members-only content (the bucket's addresses are public, so a private image needs serving through the site with an access check), video, documents such as PDFs, cropping and editing, and focal points.

## Order of work

1. Confirm the Blob store's region.
2. media's core: contract, storage port, processing, records, Publisher's asset resolver, fakes and tests.
3. Its presentation part: the picker and the library.
4. `examples/minimal-host`, `CONTRACT.md` and a release; then content-editor builds on it.
