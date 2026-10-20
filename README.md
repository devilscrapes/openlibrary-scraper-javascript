<p align="center"><a href="https://apify.com/DevilScrapes/openlibrary-books-scraper"><img src="assets/banner.svg" alt="Open Library Scraper — Book Metadata in Bulk — JavaScript example" width="100%"></a></p>

# Open Library Scraper for JavaScript and Node.js: book lookup by title, author or ISBN

<p align="center">
<a href="https://apify.com/DevilScrapes/openlibrary-books-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/openlibrary-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/openlibrary-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository calls the **[Open Library Books Scraper](https://apify.com/DevilScrapes/openlibrary-books-scraper)** from Node.js using one ES module and the official `apify-client` package. Send a search string, optionally tied to the title, author, subject or ISBN field, and get back plain objects describing works from Open Library: names, years, edition counts, ISBN arrays, publishers, a large cover URL and the e-book status. The work happens on Apify, so a route handler, a queue worker or a scheduled function only has to start a run and collect rows. Reading apps, bookshop backends and catalogue tools all get the same shape every time, which is the whole bargain; no deals with the devil required.

> **Actor:** [Open Library Scraper — Book Metadata in Bulk](https://apify.com/DevilScrapes/openlibrary-books-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [openlibrary-scraper-python](https://github.com/devilscrapes/openlibrary-scraper-python)

## 🔥 What you get

- **Works, not editions**: each row is one Open Library work, with `edition_count` telling you how many printings and translations sit behind it.
- **Point the query at one field**: `searchField` sends your text to title, author, subject or ISBN matching, or leaves it free-form with `all`.
- **A language filter**: a three-letter code such as `eng` or `spa` in `language` is passed to Open Library's search as a language filter.
- **Identifiers you can join on**: the work key, a canonical `work_url`, every ISBN-10 and ISBN-13 Open Library lists, and the numeric `cover_id`.
- **A ready cover link**: `cover_url_l` is built from the cover ID and points at the large JPEG on Open Library's cover service.
- **Reader signals where they exist**: average rating, rating count and an `ebook_access` value that says whether a scan is public, borrowable or absent.
- **Bounded subject lists**: subjects are cut at 30 per work, so one heavily tagged classic cannot bloat a CSV column.

## 🎯 Use cases

### Power an ISBN lookup endpoint

Expose `GET /books/:isbn` in Express or Fastify, start a run with `searchField: 'isbn'` and `maxResults: 1`, cache the row by ISBN and answer from the cache next time. Your users get title, author and cover from one scan of a barcode.

### Fill in cover art for a reading app

When a user adds a book by title, look it up and store `cover_url_l` next to it. The link points straight at Open Library's cover service, so your front end can render it in an `<img>` tag without hosting images yourself.

### Flag books that can be read online

Filter rows on `ebook_access` and show a 'read now' badge for `public` or `borrowable` works, linking to `work_url`. One `Set` lookup on a single field is all the logic it takes.

### Seed a recommendations service

Pull works for a few subjects on a schedule and store `subjects`, `authors` and `ratings_average` in your database. A recommender can then suggest books that share tags with the ones a reader already finished.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/openlibrary-scraper-javascript.git
cd openlibrary-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Open Library Scraper — Book Metadata in Bulk](https://apify.com/DevilScrapes/openlibrary-books-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

The integration your service needs is just the following. `call()` resolves after the run has finished, and `listItems()` hands back that run's dataset as one array of objects.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/openlibrary-books-scraper').call({"searchQuery": "isaac asimov foundation", "searchField": "all", "maxResults": 10, "language": "", "proxyConfiguration": {"useApifyProxy": false}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` reads `input.json` and sends it unchanged. This sample matches the Actor's shipped defaults: Asimov's Foundation as free text, at most 10 works, every language, direct connection, for about 21.5 cents. Only `searchQuery` is mandatory. From a request handler, set `searchField` to the kind of value the user typed and keep `maxResults` small for interactive lookups; bulk jobs can go up to 1,000.

```json
{
  "searchQuery": "isaac asimov foundation",
  "searchField": "all",
  "maxResults": 10,
  "language": "",
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `searchQuery` | string | yes | `"isaac asimov foundation"` | — |
| `searchField` | string | no | `"all"` | `all`, `title`, `author`, `subject`, `isbn` |
| `maxResults` | integer | no | `30` | — |
| `language` | string | no | `""` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

You get an array of works. Treat `openlibrary_key`, `title` and `work_url` as guaranteed and type the rest as optional. Array fields come back as empty arrays rather than null when Open Library has nothing, so `.join()` and `.length` are safe on them.

| Field | Type | Always present |
|---|---|---|
| `openlibrary_key` | string | yes |
| `title` | string | yes |
| `subtitle` | string or null | no |
| `authors` | array | no |
| `first_publish_year` | integer or null | no |
| `edition_count` | integer | no |
| `languages` | array | no |
| `subjects` | array | no |
| `isbns` | array | no |
| `publishers` | array | no |
| `cover_id` | integer or null | no |
| `cover_url_l` | string or null | no |
| `ratings_average` | number or null | no |
| `ratings_count` | integer or null | no |
| `ebook_access` | string or null | no |
| `work_url` | string | yes |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.0015 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/openlibrary-books-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

This module turns the saved rows into a small 'shelf' for a front end: only works with a public or borrowable e-book, with the first ISBN-13 picked out, sorted oldest first and written to `shelf.json`. Node.js built-ins only.

```javascript
import { readFile, writeFile } from 'node:fs/promises';

const books = JSON.parse(await readFile('results.json', 'utf8'));
const readable = new Set(['public', 'borrowable']);

const shelf = books
    .filter((book) => readable.has(book.ebook_access))
    .map((book) => ({
        id: book.openlibrary_key.replace('/works/', ''),
        title: book.subtitle ? `${book.title}: ${book.subtitle}` : book.title,
        by: (book.authors ?? []).join(', '),
        year: book.first_publish_year,
        cover: book.cover_url_l,
        isbn13: (book.isbns ?? []).find((code) => /^97[89]\d{10}$/.test(code)) ?? null,
        link: book.work_url,
    }))
    .sort((a, b) => (a.year ?? 9999) - (b.year ?? 9999));

await writeFile('shelf.json', JSON.stringify(shelf, null, 2));
console.table(shelf.map(({ title, by, year, isbn13 }) => ({ title, by, year, isbn13 })));
```

## ❓ FAQ

### Should I call Open Library's search endpoint directly instead?

For a handful of lookups you can. The Actor earns its keep when you want paging, a fixed row shape, scheduling and run history handled for you, and when Apify's dataset is where the results should live anyway.

### Does it work in a serverless function?

Small lookups finish quickly, so awaiting `call()` usually fits in a function's time limit. For bulk pulls, start the run with `.start()` and pick up the dataset from a webhook when Apify reports it finished.

### How do I type the rows in TypeScript?

`apify-client` brings its own declarations for runs and datasets. Add a `Work` interface from the output table above, with `openlibrary_key`, `title` and `work_url` as required strings, and cast `items` to `Work[]`.

### Why did a search return fewer works than `maxResults`?

Either the query genuinely matched fewer works, or Open Library answered one page with an error, which ends paging without a retry. A language filter also narrows results more than people expect.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/openlibrary-books-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Open Library Scraper — Book Metadata in Bulk on Apify Store](https://apify.com/DevilScrapes/openlibrary-books-scraper)
- [Python version of this example](https://github.com/devilscrapes/openlibrary-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Goodreads Book & Reviews Scraper](https://apify.com/DevilScrapes/goodreads-book-reviews-scraper) — reader reviews to set next to Open Library's catalogue data.
- [Crossref Works Scraper — DOIs, Authors & Citations](https://apify.com/DevilScrapes/crossref-works-scraper) — DOI metadata for journal articles and scholarly books.
- [arXiv Scraper — Search & Export Paper Metadata](https://apify.com/DevilScrapes/arxiv-papers-scraper) — preprint records when the reading list turns academic.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/openlibrary-books-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Open Library or the Internet Archive.
