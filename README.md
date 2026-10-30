# 🧩 Sync for Reddit Fixes

Morphe patches that fix **load more comments** and **Reddit galleries** in **Sync for Reddit** and
**Sync for Reddit Pro**, which broke when Reddit blocked `/api/morechildren` and `/api/info` for
third-party clients.

These patches are ported from [notBrian2/revanced-rif-fixes](https://github.com/notBrian2/revanced-rif-fixes)
(written for rif is fun) and adapted to Sync's Volley-based networking code. Use them with the Sync patches from
[wchill/patcheddit](https://github.com/wchill/patcheddit), which handle API access (Spoof client), login and
the other fixes Sync needs.

### How to use these patches

Click the button below, or manually add this repo url to Morphe Manager: https://github.com/trulow/sync-reddit-fixes

<a href="https://morphe.software/add-source?github=trulow/sync-reddit-fixes">
  <img src=".github/assets/add-to-morphe.svg" alt="Add to Morphe" height="48">
</a>

## 🩹 Patches list

<!-- PATCHES_START EXPANDED -->

<!-- Do not modify this section by hand. The patch list is generated when release.yml creates a new release. -->

| Patch | What it fixes |
|-------|---------------|
| **Fix load more comments** | Tapping "load more comments" shows *Error loading more comments*: Reddit blocked `/api/morechildren`. Sync now loads the missing comments from `/comments/{post}/_/{parent}`, the endpoint it already uses for single-comment threads, and turns the reply back into the list Sync expects. |
| **Fix Reddit galleries** | Galleries fail with *Could not load gallery*: Reddit blocked `/api/info`. Sync now loads the gallery post from `/comments/{post}`, which carries the same `media_metadata` and `gallery_data`. |

<!-- PATCHES_END -->

Both patches are enabled by default and are independent: each can be selected or deselected on its own.

### Supported apps

| App | Package | Version |
|-----|---------|---------|
| Sync for Reddit | `com.laurencedawson.reddit_sync` | `v23.06.30-13:39` (version code 23033) |
| Sync for Reddit Pro | `com.laurencedawson.reddit_sync.pro` | `v23.06.30-13:39` (version code 23033) |

This is Sync's final release. The free and Pro builds have the same code but different obfuscated names
(for example `y8/q.b()` vs `j8/q.c()`), so the patches find their targets by the source file names Sync keeps
(`OAuthGalleryRequest.java`, `OAuthCommentsMoreRequest.java`, `CommentsPresenterImpl.java`) plus strings and
method signatures, never by obfuscated names.

### Known limitations

- **Very large threads:** Reddit returns at most 500 comments per request. Deep in very large threads the last
  few top-level comments can fall outside that window. Sync then shows *Error loading more comments* and keeps
  the "load more" row, instead of spinning or removing it.
- These patches only change *where* the two requests go. Sync still needs working API access (the patcheddit
  **Spoof client** patch with your own client ID), or no request succeeds.

## 📲 Applying the patches with Morphe

You need your own copy of Sync for Reddit or Sync for Reddit Pro `v23.06.30-13:39`. This project doesn't
provide APKs.

### Morphe Manager (on your phone)

1. Add this repository as a patch source: tap **Add to Morphe** above, or in Morphe Manager add the source
   `https://github.com/trulow/sync-reddit-fixes`. (Requires a GitHub release, see *Releasing* below.) To use a
   bundle you built yourself, copy `patches-<version>.mpp` to the phone and add it as a local patch file.
2. Add the patcheddit source as well (`https://github.com/wchill/patcheddit`) for **Spoof client** and its other
   Sync patches.
3. Select your **original, unpatched** Sync APK, then select **Fix load more comments** and/or **Fix Reddit
   galleries** along with the patcheddit patches you use.
4. Patch and install.

### Morphe desktop / CLI

Download `morphe-desktop-<version>-all.jar` from
[MorpheApp/morphe-cli releases](https://github.com/MorpheApp/morphe-cli/releases) and run:

```bash
java -jar morphe-desktop-all.jar patch -p patches-1.0.0.mpp -p patcheddit.mpp -o sync-patched.apk sync.apk
```

To apply only one of the fixes, add `--exclusive -e "Fix Reddit galleries"` (or `-e "Fix load more comments"`).
Use `list-patches --patches=patches-1.0.0.mpp -p -v` to see the patches with their supported packages and
versions.

## 🛠️ Building

Requirements:

- JDK 17 or newer. Android Studio's bundled runtime works:
  `export JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"`
- The Android SDK with **platform 36** (`ANDROID_HOME` set, or `sdk.dir` in `local.properties`).
- A GitHub token with the `read:packages` scope, since Morphe publishes its Gradle plugin and patcher to
  GitHub Packages. Either add `gpr.user` / `gpr.key` to `~/.gradle/gradle.properties`, or export
  `GITHUB_ACTOR` and `GITHUB_TOKEN`. With the GitHub CLI:

  ```bash
  gh auth refresh -h github.com -s read:packages
  export GITHUB_ACTOR=<your-github-username> GITHUB_TOKEN=$(gh auth token)
  ```

Then build:

```bash
./gradlew buildAndroid
```

The bundle is written to `patches/build/libs/patches-<version>.mpp`. `buildAndroid` is required: it adds the
DEX that Morphe Manager needs on Android.

### Project layout

The layout follows [patcheddit](https://github.com/wchill/patcheddit): one extension module per app under
`extensions/`, and the patches for an app under its own `syncforreddit` package.

```
extensions/syncforreddit/                    Extension module, built to extensions/syncforreddit.mpe
  build.gradle.kts
  src/main/java/app/syncfixes/extension/syncforreddit/
    FixLoadMoreCommentsPatch.java            Builds the /comments request, converts its reply
    FixGalleriesPatch.java                   Rewrites the gallery URL, unwraps its reply
    Responses.java                           Shared Volley response helpers
  stub/                                      Compile-only Volley stubs (Sync ships Volley unobfuscated)
patches/src/main/kotlin/app/syncfixes/patches/syncforreddit/
  shared/Constants.kt                        Supported apps, versions, shared names
  fix/comments/Fingerprints.kt               Targets for "load more comments"
  fix/comments/FixLoadMoreCommentsPatch.kt   Bytecode hooks for "load more comments"
  fix/gallery/Fingerprints.kt                Targets for galleries
  fix/gallery/FixGalleriesPatch.kt           Bytecode hooks for galleries
```

The Kotlin patches only insert calls; the logic lives in the Java extension, which is merged into the app.

### How the fixes work

**Fix Reddit galleries** hooks `OAuthGalleryRequest`:

1. Its static URL builder returns `https://oauth.reddit.com/comments/{id}.json?limit=1` instead of
   `/api/info.json?id=t3_{id}`. This URL is also Volley's cache key, which Sync uses to remember the gallery
   position, so both stay consistent.
2. `parseNetworkResponse()` first replaces the response with one whose body is the first element of the
   `/comments` reply: the same `Listing` `/api/info` returned. Sync's parser and its cache handling are unchanged.

**Fix load more comments** hooks three places:

1. `CommentsPresenterImpl`'s "load more" handler passes the "more" row to the extension, which reads its
   `parent_id` (the request itself isn't given the parent).
2. `OAuthCommentsMoreRequest`'s constructor passes the post id, child ids and sort. Right before the super
   constructor, the `api/morechildren` POST is replaced by a GET of
   `/comments/{post}.json` (top level, `depth=2`) or `/comments/{post}/_/{parent}.json` (nested, `depth=10`),
   with `limit=500`, the sort, and a key to the child id list (hundreds of ids would make the URL too long).
3. `parseNetworkResponse()` turns the comment tree into morechildren's flat `{"json":{"data":{"things":[...]}}}`:
   the requested comments and everything under them, in pre-order (parents before replies, which Sync uses for
   indentation), with "more" items for deeper replies. If nothing could be loaded it returns Reddit's error
   shape, so Sync shows its normal error and keeps the row.

## 🚀 Releasing

This project keeps the template's release flow: commit to `dev` with semantic commit messages
(`feat:`, `fix:`, `chore:`), and `release.yml` builds, signs (optional), and publishes releases. Merge `dev` into
`main` for a stable release. See the
[Morphe patches template](https://github.com/MorpheApp/morphe-patches-template) for details. Before publishing,
update `source` in [patches/build.gradle.kts](patches/build.gradle.kts) and the links in this README if the
repository isn't `trulow/sync-reddit-fixes`.

## ⚠️ Disclaimer

Not affiliated with or endorsed by Laurence Dawson, Sync for Reddit, Reddit or the Morphe project. This
repository contains only original patch code, no APKs and no code from Sync. You are responsible for using your
own legally obtained copy of the app and for following the terms of the services it connects to.

## 📜 License

Sync for Reddit Fixes are licensed under the [GNU General Public License v3.0](LICENSE), with the additional
conditions under section 7 in the [NOTICE](NOTICE).
