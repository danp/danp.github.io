---
title: "Better GitHub Actions caching for Go"
date: 2025-02-18T12:23:45-04:00
---

I've been spending some time on [CI](https://en.wikipedia.org/wiki/Continuous_integration) improvements at [work](https://www.grax.com/) recently, mostly around cutting down how long things take.

As I looked to see if anything about our overall process could be improved, a couple things bothered me:

Why, if we were using a pretty standard GitHub Actions setup, were there indications that modules were being downloaded as part of every run?
Shouldn't that all be cached?

Why did it seem like there was always a delay before tests actually started running?
Shouldn't the first few packages' fast tests complete quickly?

## Current state and issues

Before getting into what I ended up changing, let's review the setup we had and break down the issues.

There are two main workflows I was concerned with: one for CI and another artifacts.

The CI workflow runs tests.
The artifacts workflow builds binaries for distribution.
We use [custom actions](https://docs.github.com/en/actions/sharing-automations/creating-actions/about-custom-actions) to cut down on some repetition but they both end up having this step:

``` yaml
steps:
  - name: Set up Go
    uses: actions/setup-go@v5
    with:
      go-version-file: go.mod
```

This uses the [setup-go action](https://github.com/actions/setup-go) to:

* [Install the Go version specified in `go.mod`](https://github.com/actions/setup-go#getting-go-version-from-the-gomod-file)
* [Cache dependencies and build output](https://github.com/actions/setup-go#caching-dependency-files-and-build-outputs)

Let's look at the second one in more detail.

setup-go doesn't use the [cache action](https://github.com/actions/cache) directly but they both use the same implementation so we can describe what setup-go is doing in its terms.

If you were to put an explicit action/cache step in to replicate what setup-go does, it would look something like:

``` yaml
steps:
  - name: Cache Go dependencies and build output
    uses: actions/cache@v4
    with:
      path: |
        ~/go/pkg/mod # actually uses `go env GOMODCACHE` output
        ~/.cache/go-build # actually uses `go env GOCACHE` output
      key: setup-go-Linux-x64-ubuntu24-go-1.23.6-${{ hashFiles('go.sum') }}
```

(some items in `key` made static for brevity)

When saving to the cache, this step generates a key based on:

* OS and architecture
* Go version
* A hash of the `go.sum` file

Since [cache items are immutable](https://github.com/actions/cache/blob/main/tips-and-workarounds.md#update-a-cache), that means a new cache item will be generated when one or more of those things changes.

For example, if you [updated to use Go 1.23.4](https://go.dev/doc/devel/release#go1.23.minor) around December 3rd when it came out and didn't change anything else that influenced the key until Go 1.23.5 came out in January you would have been using the same cache item that whole time.
A build that happened January 8th would have used the same cache item as one that started December 16th, regardless of how much your code had changed in the meantime.

For smaller projects that's probably not a big deal but for larger ones it can add up.
It seemed to be for ours!

We were running into another issue related to cache item immutability.

When we did make a change that led to a cache item, our faster-running artifacts workflow was sneaking in and saving a cache item based on what it was doing.
When the CI workflow finished it saw the cache key it wanted to save already existed and carried on.

The artifacts workflow for the most part runs `go build ./cmd/...`.
Our CI workflow runs something like `go test -race -count 1 ./...`.

That meant our cache was missing build output that could help build our tests[^1], especially for the [race detector](https://go.dev/doc/articles/race_detector) enabled by `-race`.
That explained the delay before our first few fast-testing packages produced any output.

The cache was also missing all the modules needed by everything CI does which explained every CI run starting with some download output.

Now that the issues were understood they could be fixed!

## The new setup

I wanted a new setup that would:

* Cache all needed modules
* Cache build output in a way that kept things fresh
* Not let the cache grow too big
* Ensure the more involved CI workflow saved cache items, not the artifacts workflow
* Only save to the cache when running CI on the `main` branch
* Let CI and artifacts runs on any branch use the cache

## Cache saving

Let's start with cache saving.

Near the end of the CI workflow, we now have this:

``` yaml
steps:
  - name: Trim Go cache
    if: ${{ github.ref == 'refs/heads/main' }}
    shell: bash
    # As the go command works, it either creates build cache files or touches
    # ones it uses at most once an hour. When it trims the cache, it trims
    # files that have not been modified/touched in 5+ days.
    # To keep our saved cache lean, trim all files except ones that were just
    # created/touched as part of this run.
    run: |
      find ~/.cache/go-build -type f -mmin +90 -delete

   - name: Set Go cache date
     shell: bash
     run: echo "GO_CACHE_DATE=$(date +%Y-%m-%d)" >> $GITHUB_ENV

  - name: Save Go cache
    if: ${{ github.ref == 'refs/heads/main' }}
    uses: actions/cache/save@v4
    with:
      # Caches both the downloaded modules and the compiled build cache.
      path: |
        ~/go/pkg/mod
        ~/.cache/go-build
      # Save to eg Linux-go-$hash-YYYY-MM-DD to keep the cache fresh
      key: "${{ runner.os }}-go-${{ hashFiles('go.mod') }}-${{ env.GO_CACHE_DATE }}"
```

The `Set Go cache date` step gives us a `GO_CACHE_DATE` environment variable with the current date.
The `Save Go cache` step saves a cache item to a key based on OS, the hash of `go.mod`, and `GO_CACHE_DATE`.

That means whenever a cache item is saved it'll end with the `GO_CACHE_DATE` value, such as `2025-02-18`.
Because cache items are immutable, if a cache item already exists with the same `GO_CACHE_DATE`, saving will be skipped.

If an existing cache was used for this run, before saving, the `Trim Go cache` step trims build output that was not created or used by the CI run that is completing.
There is [a proposal](https://go.dev/issue/69879) around making the go command's cache trimming configurable but this seems good enough for our purposes.

Finally, to ensure all needed modules end up in any saved cache items, we have this near the top of the CI workflow:

``` yaml
steps:
  - name: Download Go modules
    run: go mod download
    shell: bash
```

[`go mod download`](https://go.dev/ref/mod#go-mod-download) will pre-fill the module cache with everything the main module needs.
If everything is already present, it does nothing (and is fast).

Since only the CI action saves the cache that means the artifacts workflow can't leave us with an ineffective cache item anymore.

All this leaves us with a cache that is:

* Pre-filled with all modules the main module needs
* Updated every day or so with the use of `GO_CACHE_DATE`
* Trimmed before saving to only contain relevant build output, keeping size down
* Only saved by CI runs on `main`

## Cache restoring

Now, restoring the cache.

Near the top of both the CI and artifacts workflows, we have:

``` yaml
steps:
  - name: Restore Go cache
    uses: actions/cache/restore@v4
    with:
      path: |
        ~/go/pkg/mod
        ~/.cache/go-build
      # always grab from the restore-keys pattern below,
      # like Linux-go-$hash-YYYY-MM-DD as saved by CI
      key: nonexistent
      restore-keys: |
        ${{ runner.os }}-go-${{ hashFiles('go.mod') }}-
```

The trick here is that using `key: nonexistent` means restoring will always fall back to using [`restore-keys`](https://github.com/actions/cache#inputs).
The `restore-keys` prefix contains everything before the `GO_CACHE_DATE` value that is used to save.

This means all both the CI and artifacts workflows, on any branch, can load a cache item that was saved by a recent CI run on `main`.

For example, if the last CI run on `main` saved `Linux-go-abcd-2025-02-18`, any subsequent CI or artifacts run will use that item.

All this leaves us with a cache restore setup that:

* Both the CI and artifacts workflows, on any branch, can use
* Prefers fresh cache entries saved by CI on `main`

## Conclusion and possible improvements

All this together has helped our Go caching greatly.

Your results may vary, of course, but rolling these changes out cut about a minute and a half off our CI run time.
And it was nice to see those initial fast tests report something almost instantly.

Perhaps this should be integrated into the setup-go action somehow.

Either way a possible improvement over `GO_CACHE_DATE` could be to judge cache churn by how many build output files are created/touched during a CI run.
If many are created and few are touched, it probably indicates a new cache item is warranted.

Happy caching!

[^1]: Since we use `-count 1` to always run our tests I'm only concerned about helping our tests build faster. There is [an open issue](https://go.dev/issue/58571) related to getting reliable Go test cache hits with GitHub Actions.
