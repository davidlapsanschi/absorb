# Absorb Dev (this fork)

`absorb-dev` is our own build of [pounat/absorb](https://github.com/pounat/absorb): his code at a release we picked, plus the changes below. We don't follow his releases automatically. When David asks, we compare his new releases with this list and decide together whether to move our changes onto his newer code.

Base: **1.12.1-305** (synced 2026-11-14).

## Our changes

Each entry: what was wrong, our commit, and how to tell whether his newer code still needs it. "Upstream" means his `main`.

### 1. Highlights made in other readers land on their words
- **Problem:** highlights from KOReader or BookOrbit's web reader showed in the Highlights tab but weren't drawn, or opened at the chapter start. Their CFIs follow the EPUB spec. epub.js reads `.html` XHTML chapters as HTML and counts text after a leading element (e.g. Eye of the World's `<a id="page_N"/>` page anchors) differently, so the CFI points at other words.
- **Fix:** after highlights sync, `anchorHighlights` (epubView.js) checks each one against its words, finds them nearby if the CFI misses, and saves the corrected CFI on the phone.
- **Commit:** "Find highlights made in other readers by their words" (was PR #464, his issue #463).
- **Still needed?** Yes, unless upstream's epub.js now resolves spec CFIs correctly. Test: a Kindle/KOReader highlight in Eye of the World is drawn on the right words in Absorb. In the code: upstream has no `anchorHighlights` and still reads `.html` chapters as HTML.

### 2. Read along uses the right copy of a book read before as a plain EPUB
- **Problem:** a phone that had a book's plain EPUB cached used it as the read-along copy, so read along lit nothing until the app storage was cleared. Saved timings were also used when the server said the book no longer had a read along.
- **Fix:** a reader copy notes which server file it came from; a narrated EPUB copy without that note is checked for SMIL files or fetched again. Saved timings stand in only while the server can't be reached.
- **Commit:** "Keep read along working on BookOrbit books whose plain EPUB was read before…" (was PR #465).
- **Still needed?** Yes, if upstream's `ebook_cache.dart` still keys a copy by version only (no source file id). Test: read a book's plain EPUB, then add its read along in BookOrbit. Read along should light sentences without clearing storage.

### 3. Read along and Find in ebook work offline
- **Problem:** offline read along needs timings saved on the phone, but they were saved only when the reader was opened online. Find in ebook asked only the server. A missing reader copy of a narrated EPUB was re-downloaded with all its audio (hundreds of MB).
- **Fix:** downloading an audiobook with a matched read along also saves its timings (missed ones are caught up once per run), Find in ebook uses saved timings offline, and book details are read before re-fetching a reader copy so it comes without audio.
- **Commit:** "Keep read along and Find in ebook working offline…" (was PR #465).
- **Still needed?** Yes, if upstream's `download_service.dart` doesn't save narration timings with the audiobook download. Test: download an audiobook that has a read along, never open the reader, go to airplane mode, open the reader. Read along should follow.

### 4. Word-by-word read along for narrated EPUBs
- **Problem:** with a BookOrbit read along (EPUB3 media overlay), read along lit whole sentences only, and the Word/Sentence choice was hidden for it.
- **Fix:** with Word, the words of the lit sentence still to come are dimmed with a CSS custom highlight (`absorb-mo-ahead`), so the page markup and CFIs stay unchanged. BookOrbit read alongs made with "Sentence" only time sentences, so a word's time is its share of the sentence's text: start and end are exact, the middle is evenly spread.
- **Commit:** "Follow narrated books word by word in read along…" (was PR #466).
- **Still needed?** Yes, if upstream's reader still hides the Word/Sentence setting for narrated books (`!_useNarration` next to the e-ink check) or has no word dimming for media overlays.
- **Sync note:** 305 changed the same lines in `_narrationTick` (headphone offset, sentence mode). Keep his `heard` calculation and add our `_narrationShare(...)` line after it.

### 5. Find in ebook marks the sentence in narrated books
- **Problem:** on a narrated book, Find in ebook went to the right page but didn't mark the sentence, while transcribed books flash the found passage.
- **Fix:** the found sentence flashes once (`_flashNarrationSentence`, `__absorbMO.cfi`). Not on page turns or read along, and not over a saved highlight.
- **Commit:** "Mark the sentence Find in ebook lands on in narrated books…" (was PR #467).
- **Still needed?** Yes, if upstream doesn't flash the sentence after Find in ebook on a book with a BookOrbit read along.

### 6. Swiping a page works on Android
- **Problem:** after a swipe, read along pulled the page straight back, and swipes didn't save the reading position (broken since 304). In `loadBook` (epubView.js), `var scrolledFlow = …` hides the `scrolledFlow()` function. The touch setup threw before adding its touchend listener, so swipes never reached the app.
- **Fix:** drop the variable and use the function.
- **Commit:** "Stop read along pulling the page back after a swipe".
- **Still needed?** Yes, while upstream's epubView.js still has `var scrolledFlow =` inside `loadBook` (it does in 305).

### 7. Page turns mid-sentence only when the voice gets there, in every chapter
- **Problem:** 305 turns the page mid-sentence once the voice reaches the part on the next page. But `_narrationHeardSpot` subtracted the clip's begin, which counts from the start of its own audio file. A read along EPUB has one file per chapter, so past the first chapter every sentence crossing a page turned the page as soon as it started.
- **Fix:** measure from `n.startOf(clip)` (the clip's start in the book).
- **Commit:** "Turn the page mid-sentence only once the voice reaches the next page, in every chapter".
- **Still needed?** Yes, while upstream's `_narrationHeardSpot` uses `heard - clip.begin`.

## Considered, not built
- **Real word timings:** BookOrbit can make a read along with "Word" granularity instead of "Sentence", giving each word its own timing from the audio. That would make word-by-word follow the narrator exactly. It needs each read along remade (hours per long book) and the reader taught to light the sentence plus the current word from word clips. Try it on a short book first.

## Syncing with upstream (only when David asks)
1. **Report first, build nothing:**
   - His releases since our base: a short summary of what's new that this branch doesn't have, so David can judge whether it's worth it.
   - New bug reports against those releases.
   - For each entry above: still needed, fixed by him (drop it), or changed so it needs rework.
2. **After David's go**, start from upstream `main` and cherry-pick our commits that are still needed (`git log upstream/main..absorb-dev`), then this file last. Update the base line and any entry that changed.
3. **Check:** `flutter analyze lib` shows 0 errors (errors under `packages/*/example` are upstream noise). Build on a clean checkout, then restore `lib/l10n/` (the build regenerates it). Check the APK contains each change: the epubView.js changes are a plain asset in it; the Dart ones show as strings inside `lib/arm64-v8a/libapp.so`.
4. **GitHub:** when his commits touch `.github/workflows/`, a token without the `workflow` scope can't sync or push. David clicks **Sync fork** on GitHub first, then `git push --force-with-lease origin absorb-dev`.
5. **Build:** `flutter build apk --release --flavor dev --split-per-abi --target-platform android-arm64`. Use the same signing key every time, kept outside this repo, so the app installs as an update.
