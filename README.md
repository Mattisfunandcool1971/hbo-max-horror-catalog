# HBO Max Horror Catalog

A reconstructed list of horror films in HBO Max's public movie sitemap, matched against IMDb metadata.

## Why this project exists

HBO Max exposes a public movie sitemap containing a large portion of its movie catalog, but the sitemap is not especially useful for browsing. At the time this project was assembled, the sitemap contained **2,325 movie entries**. Each entry supplied a displayed title and a Max movie URL/UUID, but it did **not** provide the metadata needed to answer a simple question such as:

> What horror movies are actually in the HBO Max movie catalog?

There was no genre field, no release year, and no convenient way to distinguish identically titled films from one another using the sitemap alone.

The normal Max interface is designed for recommendations and curated browsing rather than for inspecting the full underlying catalog. That makes it difficult to see older, obscure, international, cult, or otherwise less-promoted films that may still be present in the archive.

This project therefore treats the Max sitemap as a catalog-membership source and reconstructs missing metadata by matching those entries against IMDb's downloadable non-commercial datasets.

The goal is not to claim a perfect or exhaustive list. The goal is to produce a **conservative, reproducible set of horror titles that can be identified with high confidence without guessing through ambiguous title collisions**.

## Result

The Max sitemap snapshot used here contained:

- **2,325 HBO Max movie entries**
- **2,325 unique Max UUIDs**
- **2,318 unique displayed titles**
- **7 duplicated displayed titles** represented by different Max UUIDs

After matching, normalization, ambiguity checks, and horror-specific screening, the project retained **95 high-confidence HBO Max catalog entries classified as Horror**.

Because Letterboxd represents films rather than Max-specific catalog editions, those 95 Max entries collapse to **93 unique films** in the Letterboxd-ready CSV. For example, an ASL presentation and an unrated cut can correspond to the same underlying film.

The repository currently includes:

- `HBO_Max_Horror_Letterboxd_Import_with_Synopses.csv` — 93 unique films with title, release year, and a short synopsis suitable for Letterboxd list import.

## Data sources

### HBO Max

The starting point was HBO Max's public movie sitemap:

`https://www.hbomax.com/sitemap/movies`

The sitemap supplied the current catalog entries used for this project. A typical entry contains a Max movie UUID, a movie URL, and a displayed title.

The sitemap did **not** supply genre or release-year metadata.

### IMDb

IMDb's downloadable non-commercial datasets were used to reconstruct metadata:

- `title.basics.tsv.gz`
- `title.akas.tsv.gz`

`title.basics` provides IMDb title IDs, primary and original titles, title type, release year, runtime, and genre labels.

`title.akas` provides alternate, regional, and translated titles. This was especially useful when the title displayed by Max differed from IMDb's primary title.

The large IMDb source datasets are **not redistributed in this repository**. They can be obtained directly from IMDb.

## Methodology

### 1. Extract the Max catalog

The HBO Max sitemap HTML was parsed locally.

For every sitemap entry, the following information was retained:

- Max UUID
- displayed Max title
- Max movie URL

The UUID was important because the displayed title alone is not a unique identifier. Different films, cuts, accessibility versions, or remakes can share the same visible title.

No duplicate Max UUIDs were removed.

### 2. Preserve duplicate displayed titles

The initial extraction produced 2,325 distinct Max UUIDs but only 2,318 unique displayed titles.

Seven displayed titles occurred more than once:

- A Nightmare on Elm Street
- A Very Harold & Kumar 3D Christmas
- Belle
- Grey Gardens
- Mortal Kombat
- Shadows
- Supergirl

These were kept as separate catalog records rather than being collapsed by title.

### 3. Normalize titles

Max and IMDb do not always format the same movie title identically. A deterministic normalization pass was therefore applied before matching.

Normalization included:

- Unicode normalization
- straightening curly quotation marks and apostrophes
- normalizing dashes and whitespace
- case folding
- treating `&` and `and` consistently
- removing punctuation for comparison
- stripping surrounding quotation marks
- recognizing explicit year hints when Max supplied them
- accounting for presentation labels and edition suffixes such as:
  - `(with ASL)`
  - `(ASL)`
  - language/audio labels
  - `Extended Edition`
  - `Extended Cut`
  - `Unrated`
  - `Director's Cut`
  - `Ultimate Edition`
  - `Singalong Version`

The purpose of normalization was to reconcile formatting differences, not to force unlike titles into a match.

### 4. Exact-match against IMDb basics

Normalized Max titles were compared against both:

- IMDb `primaryTitle`
- IMDb `originalTitle`

The search initially included IMDb records with title types relevant to material that can appear in Max's movie sitemap:

- `movie`
- `tvMovie`
- `video`
- `short`
- `tvSpecial`

This broader set was necessary because Max's sitemap labeling does not map perfectly onto IMDb's type taxonomy. A Max "movie" entry can sometimes correspond to a special, short, or video record in IMDb.

Only unique matches, or matches disambiguated by an explicit Max year hint, were automatically accepted at this stage.

If more than one IMDb record matched the same normalized title, the case was retained as ambiguous rather than guessed.

### 5. Use IMDb alternate titles

For unresolved titles, IMDb's `title.akas` dataset was used to search alternate and regional names.

U.S. alternate titles were treated as especially useful evidence because the Max catalog examined here is the U.S. catalog.

This resolved additional entries where Max's displayed title differed from IMDb's primary title.

Again, an alternate-title hit was not accepted if multiple plausible IMDb records remained.

### 6. Limit fuzzy matching

A broad fuzzy-title comparison against all of IMDb proved both computationally expensive and methodologically noisy.

The workflow was therefore redesigned to avoid all-against-all matching.

For the remaining unmatched Max titles:

1. distinctive title tokens were extracted;
2. IMDb was streamed once;
3. only IMDb records sharing meaningful title tokens were retained as candidates;
4. fuzzy similarity was calculated only within that reduced candidate pool;
5. only very high-similarity matches with clear separation from competing candidates were automatically accepted.

This greatly reduced computation while also avoiding many spurious title matches.

### 7. Invert the problem around Horror

At this point, resolving every one of the 2,325 Max entries was unnecessary.

The research question was not "What is every movie in the sitemap?" but "Which entries can be identified as Horror?"

The workflow was therefore inverted:

- retain already-resolved entries whose IMDb genre includes `Horror`;
- inspect ambiguous records only when at least one plausible IMDb candidate is Horror;
- build a local Horror-only subset of IMDb for screening true unknowns;
- defer unrelated documentaries, specials, comedies, and other titles that did not produce meaningful Horror evidence.

This avoided spending substantial effort positively identifying material irrelevant to the project.

### 8. Check exact-title collisions

A major source of false positives was the existence of unrelated IMDb records with the same title.

For example, a well-known non-horror feature can share its title with an obscure horror short. Merely finding *a* Horror record with the same title is therefore not enough.

For provisional Horror matches, the project made another pass through IMDb and collected **all exact normalized title matches**, including Horror and non-Horror records.

Cases were then separated into:

- exact-title records that consistently supported Horror;
- titles mixing Horror and non-Horror candidates;
- cases where the apparent Horror match depended on assuming one IMDb title type was more likely than another.

The project deliberately rejected the shortcut of assuming that an IMDb `movie` record must be the correct Max entry while a `short`, `video`, or `tvSpecial` record must be wrong. Max's sitemap itself contains material corresponding to several IMDb title types, so that assumption could create false confidence.

### 9. Define the high-confidence set conservatively

The final working set consists of:

- **84 Max entries with unique IMDb matches classified as Horror**, plus
- **11 additional entries where the exact-title evidence supported Horror without depending on a speculative title-type preference**

This produced the final count of:

**95 high-confidence HBO Max Horror entries**

These correspond to **93 unique films** after collapsing Max-specific duplicate presentations for the Letterboxd version.

The list is intentionally conservative.

A title was omitted when the available sitemap and IMDb evidence could not identify the Max catalog object confidently enough.

## What "high confidence" does and does not mean

"High confidence" means that the title/metadata relationship was supported strongly enough by the Max sitemap and IMDb data that the project did not need to choose arbitrarily among conflicting candidates.

It does **not** mean:

- the list contains every horror film currently on Max;
- IMDb's genre taxonomy is definitive;
- every Max catalog entry has been fully resolved to an IMDb ID;
- the catalog will remain unchanged over time.

False negatives are much more acceptable here than false positives. When evidence was ambiguous, the title was generally left out.

## Synopses

The CSV includes a short, spoiler-light synopsis for each unique film.

These descriptions are intended as browsing aids rather than archival metadata. They are concise paraphrases, not copies of IMDb plot-summary text. Recent, obscure, or potentially ambiguous films were checked against public reference sources before their descriptions were finalized.

## Limitations

### The Max catalog changes

Streaming catalogs are dynamic. Films are added and removed, and Max may alter its sitemap structure.

The count of **2,325 entries** describes the sitemap snapshot used for this project, not a permanent total.

### IMDb genre labels are imperfect

IMDb can assign multiple genres to a film, and genre classification is inevitably contestable around horror-adjacent material such as:

- monster movies
- dark fantasy
- supernatural thrillers
- kaiju films
- horror-comedies
- exploitation films

For consistency, the primary inclusion signal was whether the matched IMDb metadata contained the `Horror` genre.

### Same-title collisions remain difficult

Without release year or other metadata supplied directly by Max, some Max UUIDs cannot be safely mapped to one of several IMDb records sharing the same title.

Those uncertain cases were generally excluded rather than resolved by popularity, title type, or intuition.

### This is not an official HBO Max or IMDb dataset

This is an independent reconstruction made from publicly accessible catalog information and IMDb's non-commercial datasets.

The project is not affiliated with, endorsed by, or maintained by HBO, Warner Bros. Discovery, Max, IMDb, or Letterboxd.

## Why publish it?

The practical reason is simple: a subscriber should be able to answer a question like "What horror movies are actually in this service's movie archive?"

The Max interface is useful for recommendations, but it is not designed to expose the complete catalog in a systematic way. The sitemap reveals that the archive is much larger and stranger than the normal browsing interface suggests, including older films, international titles, cult movies, studio horror, and obscure material that may otherwise be difficult to discover.

Publishing the reconstructed list makes that hidden portion of the catalog easier to browse, check, improve, and share.

## File format

The Letterboxd-ready CSV contains:

| Column | Description |
| --- | --- |
| `Title` | Film title used for Letterboxd matching |
| `Year` | Release year |
| `Review` | Short spoiler-light synopsis, imported by Letterboxd as a list note |

## Reproducibility and future improvements

A future version of the project could improve coverage by adding a reliable source of metadata tied directly to each Max UUID, especially release year or external IDs.

That would make it possible to resolve many of the same-title cases that were intentionally excluded here.

Other possible extensions include:

- repeating the sitemap extraction periodically to track additions and removals;
- reconstructing other genres;
- adding IMDb IDs to a research-oriented version of the dataset;
- comparing the sitemap against what Max exposes through its normal genre browsing interface;
- documenting titles present in the sitemap but difficult to surface through ordinary browsing.

Contributions, corrections, and reproducible improvements are welcome.
