# Recontext

Find earlier appearances of an image and help a journalist check whether its current caption misrepresents the event, date, or place.

**Status:** project brief. No application or provider integration has been implemented or tested yet. Start with one image and one claim; evaluate that workflow before adapting it to batches.

## 1. Information sources and how we will access them

Provider documentation reviewed on **October 26, 2026**. Documented capability is not a successful integration test; credentials, account access, quotas, costs, and retrieval quality still need checking.

| Source | Information it can contribute | Planned access | Role and limits |
| --- | --- | --- | --- |
| **Google Cloud Vision Web Detection** | Matching image URLs, pages containing full or partial matches, and visually similar images | Backend request to `POST https://vision.googleapis.com/v1/images:annotate` with `WEB_DETECTION`. Send image bytes as base64; use Google Cloud credentials, enable the API, and configure billing. | Proposed first automated image-search source. Similarity is a lead, not proof that two images show the same event. Results do not establish capture dates. |
| **Google Fact Check Tools API** | Previously published fact-checks associated with an image or claim | Backend request to `GET https://factchecktools.googleapis.com/v1alpha1/claims:imageSearch`, with `imageUri` and an API key. Text queries can use `claims:search`. | Supplemental lookup for existing investigations. This searches fact-checked claims, not the entire web. An empty result says nothing about authenticity. |
| **TinEye API** | Matching images, referring pages, and crawl dates | Authenticate backend requests with the `x-api-key` header. Use the documented URL-search or image-upload method at `https://api.tineye.com/rest/`. | Optional additional search source after comparing coverage and cost. Real searches require a paid search bundle. The free sandbox returns dummy results and cannot validate retrieval quality. |
| **InVID / WeVerify / vera.ai** | Manual reverse-search workflows, image inspection, and video keyframe extraction | Use the browser extension during research and evaluation; retain useful source links in a case. | A toolkit, not a single public search API we can assume is available to our backend. Its external services have their own access requirements. Video support is a later extension. |
| **Original webpages, image captions, and source posts** | Publication dates, location descriptions, surrounding text, credits, and the actual claim | Open candidate sources and, where accessible and permitted, retrieve their public pages. Preserve the URL, relevant excerpt, access time, and how each date was obtained. | Evidence for reviewing a candidate match. Mark inaccessible pages explicitly. Keep the claimed event date separate from the page publication date and our retrieval time. |
| **Google Lens / TinEye website** | Additional matches found by a researcher | Manual browser searches and user-supplied evidence links | Baseline and fallback for coverage gaps. The first version does not depend on scraping consumer search interfaces or automating logged-in social accounts. |

### Provider references

- Google Cloud Vision: [Web Detection and request examples](https://docs.cloud.google.com/vision/docs/detecting-web), [pricing](https://cloud.google.com/vision/pricing).
- Google Fact Check: [API setup](https://developers.google.com/fact-check/tools/api/), [image search](https://developers.google.com/fact-check/tools/api/reference/rest/v1alpha1/claims/imageSearch).
- TinEye: [API example and response fields](https://help.tineye.com/article/182-using-the-tineye-api-to-determine-if-an-image-is-a-stock-photo), [authentication and errors](https://help.tineye.com/article/293-tineye-api-error-messages), [purchasing searches](https://help.tineye.com/article/275-signing-up), [sandbox limitations](https://help.tineye.com/article/166-testing-integration-with-the-api-sandbox), [meaning of “first found”](https://help.tineye.com/article/248-what-does-tineye-first-found-on-mean).
- InVID: [current plugin repository](https://github.com/AFP-Medialab/verification-plugin), [tool overview](https://www.invid-project.eu/tools-and-services/invid-verification-plugin/). The older `AFP-Medialab/invid-verification-plugin` repository is archived.
- Google Lens: [image-context workflow](https://blog.google/products-and-platforms/products/search/about-this-image-google-search/).

### Initial access decisions

Start with **Cloud Vision Web Detection** as the proposed automated retrieval provider. Its documented matching-page results fit the single-image workflow, and image bytes can be submitted without publishing an upload at a public URL. This choice remains provisional until a small real-image evaluation establishes coverage and cost.

Add **Google Fact Check** as a separate source of prior investigations. Its image endpoint requires a publicly reachable image URL. Initially use images already published publicly; do not silently make a private upload public. Public URLs can also fail to fetch. Keep original-language evidence when translating results for a Russian-speaking reviewer, and avoid restricting retrieval to Russian sources alone.

Evaluate **TinEye** if it supplies useful matches the initial provider misses. Its crawl dates indicate when TinEye observed an image, not when the photograph was taken. Purchases and automatic top-ups are not part of repository setup.

Provider credentials will stay on the backend. Before processing real submissions, disclose which providers receive the image and define deletion/retention behavior. Keep uploads, credentials, and research evidence outside the public repository. Source material remains attributable to its publisher; a link to an investigation does not grant permission to redistribute its media.

## 2. Problem

A genuine photograph can become misleading when presented as evidence of a different event. An older synthetic image can also be recycled as if it depicts breaking news. The task is to establish the relationship between the image and its current claim.

A researcher currently has to search for earlier appearances, distinguish the same image from merely similar scenes, inspect dates and captions, follow original sources, and assemble a defensible explanation. Those steps are spread across search tools, browser tabs, and notes.

The initial audience is independent fact-checkers and journalists reviewing Russian-language current-event posts. The proposed value is less time assembling and reviewing evidence. Demand and time savings have not yet been measured.

## 3. Proposed solution

For **one image and one accompanying claim**, return an evidence report that helps a person answer:

> Has this image appeared in a context that conflicts with what this post claims?

The report should show the submitted claim, candidate image matches, earlier source links, relevant captions, separately labeled dates, and unresolved questions. Every factual explanation must point to its supporting source.

A useful report might say that a post associates an image with current protests in France, while a matching earlier source describes an incident in Sweden in 2021. The reviewer still checks the image correspondence and whether the current post labels the image as archival.

Finding an image on an older page is evidence to investigate. It is not, by itself, proof of its capture date, the publisher's intent, or the falsity of every statement in a post. No matches, provider errors, and inaccessible evidence must remain visible rather than becoming a “verified” result.

## 4. Approach 1: check one image and its claim

### Input

- One image file or public image URL, including a manually selected video frame if needed.
- The accompanying caption or claim, supplied or confirmed by the researcher.
- The source-post URL and its publication time, if known.
- The claimed event, location, and date, where stated; missing values remain unknown.

### Workflow

1. **Preserve the submission.** Retain the original image and claim. Record any crop used for searching separately, so a screenshot's surrounding text is not lost.
2. **Retrieve candidates.** Query the configured image-search provider; optionally look up existing fact-checks. Record provider failures separately from successful searches with no matches.
3. **Inspect the matching sources.** Collect source URLs, candidate images, captions, publication dates, and relevant passages. The reviewer can supply evidence links when pages cannot be fetched automatically.
4. **Compare context.** Highlight potential time or location conflicts with source citations. Treat visual similarity and extracted dates as candidates for review. Preserve an original passage alongside any translation.
5. **Produce an evidence report.** Keep candidate matches, confirmed evidence, prior fact-checks, and the reviewer's conclusion distinguishable. Provide a simple export with the claim, sources, dates, and outstanding questions.

The first implementation should be one straightforward end-to-end path using one retrieval provider. An LLM is optional for organizing or translating retrieved text; it cannot supply an image's history from memory. UI framework, language, and deployment choices remain open.

### Check the approach before scaling

First verify provider access with real requests. Then assemble a small pilot set, provisionally **12 cases**: four known misleading reuses, four legitimate archival or illustrative uses, and four difficult or unresolved examples. Include crops and screenshots. The articles below are starting references, not a completed dataset.

Compare the application with manual reverse-image search on the same cases. Record:

- Whether it retrieves a useful source, and whether that source actually contains the matching image.
- Whether the date and location claims are supported by inspectable evidence.
- False alarms on legitimate reuse, missed leads, and failures caused by inaccessible sources.
- Time to an inspectable evidence report, reviewer effort, and provider calls/cost per case.

Report actual counts and unresolved cases. This pilot is too small to establish general accuracy. Before expanding, require inspectable citations for every reported conflict, explicit handling of unavailable evidence, and a repeatable benefit over the manual baseline.

Keep **rediscovery** and **new-lead discovery** separate in evaluation. Finding a completed fact-check is useful, but it does not show that the tool could have discovered the case before publication. For historical discovery tests, restrict eligible evidence to what existed before the fact-check and record that cutoff.

## 5. Approach 2: adapt the verified workflow to batches

After implementing and evaluating Approach 1, reuse the same check for a small researcher-supplied collection of images, captions, and post URLs. The verification task stays the same; the new work is coordinating runs and making results useful together.

Add only what batching needs:

- Track each item independently, with progress, cancellation, and recoverable failures.
- Bound concurrency, retries, and per-batch provider spending.
- Reuse image-retrieval results for identical images where provider terms allow, while evaluating each distinct caption separately.
- Group repeated or near-duplicate media for review without silently treating similarity as identity.
- Present a shortlist ranked by the strength of the available evidence and apparent conflict, with links into each individual report.

A batch of repeated images should not become a batch of duplicate leads. A reused image with two different captions still represents two contextual checks.

Continuous monitoring of social platforms would be a separate later decision, dependent on platform access and demonstrated user need. It is not required to establish whether batch review is useful.

## 6. Reference investigations

These are published investigations to study and independently reproduce where possible. Their conclusions have not been reproduced by this project.

| Investigation | What it teaches us | Scope |
| --- | --- | --- |
| [Swedish footage presented as French protests](https://factcheck.kz/articles/snimok-iz-shvecii-vydayut-za-foto-protestuyushhih-vo-francii) | A genuine image can have the wrong location and date attached. The article traces it to footage from Gothenburg in 2021. | Primary single-image reference case. |
| [Images of the burning Eiffel Tower](https://factcheck.kz/articles/ii-izobrazheniya-goryashhey-eyfelevoy-bashni-rasprostranilis-v-seti) | Earlier appearances can reveal a misleading timeline even when synthetic-media analysis is a separate question. | Test earlier-appearance retrieval; do not reproduce detector scores as calibrated truth probabilities. |
| [Older US protest videos presented as recent](https://factcheck.kz/articles/proshlogodnie-video-protestov-v-ssha-vydali-za-svezhie) | Correct country and protest name can still conceal a wrong year. | Manually selected frames first; automated video handling later. |
| [Zelensky and the claim about 25% of Ukraine](https://factcheck.kz/articles/zelenskiy-ne-soglashalsya-otdat-25-territorii-ukrainy-radi-zaversheniya-voyny) | Verifying an interpreted statement requires the full interview and linguistic context. | Outside the initial image-context workflow. |

## 7. Milestones

- [x] Describe the problem, proposed sources, and phased approach.
- [ ] Confirm provider credentials, quotas, and acceptable costs with real requests.
- [ ] Prepare the pilot cases and measure the manual baseline.
- [ ] Implement Approach 1: one image, one claim, one evidence report.
- [ ] Evaluate Approach 1 and publish measured results and limitations.
- [ ] Adapt the same workflow to Approach 2: small batches and a review shortlist.
- [ ] Evaluate whether batch review produces useful leads with manageable review effort.

The first useful release is an evidence-gathering aid that individuals can try independently. Automatic truth verdicts, interview interpretation, AI-image detection, and whole-platform crawling are outside that first release.
