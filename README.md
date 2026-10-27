# The Actual action

The action reviews a pull request by running it. It runs as the last step
of the job that already builds and tests your repository. The review works
in what that job's steps left on the runner: the toolchain on `PATH`, the
dependencies and build outputs in the checkout's ignored directories, and
the test reports. The action installs and fetches nothing for your
code.

## Add it to the job that runs your tests

Add one step after your own build and test steps:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@<sha>
        with:
          fetch-depth: 0
          persist-credentials: false
      - run: <your own install, build, and test steps, unchanged>
      - uses: nox-systems/actual-action@v0
```

The step runs only when the steps before it passed, so a change whose own
tests fail is never reviewed. Each setting in the snippet does the
following:

- `fetch-depth: 0` fetches the merge base, because the review extracts
  both the base and the head tree from the checkout's repository.
- `persist-credentials: false` keeps the job's token out of `.git`. The
  review extracts the trees without `.git` either way.
- `id-token: write` lets the job request the OIDC token that the action
  exchanges for a run token. Without it, the step fails with one line
  that names the permission.
- The action needs no other permission, because it never uses the job's
  GitHub token: Actual reads the pull request through its GitHub App.

A pull request from a fork gets no OIDC token, so the action prints one
line and exits 0, and the pull request's checks stay green. It does the
same for a job Dependabot started that GitHub gave no token, and for a
job whose event is not a pull request, such as a push to your default
branch.

### The model

Actual supplies the model. The review's model calls go to Actual's API
under the run's token, and the worker calls the model provider with
Actual's own key, so you set no model key and none reaches your runner.
The model is the one your org chose in Actual's settings: Claude Opus 5.5
unless an owner chose another.

The `model` input asks for a model in the workflow. It takes `opus-5.5` or
`fable-5.1`, and also the full ids `claude-opus-5-5` and
`claude-fable-5-1`:

```yaml
      - uses: nox-systems/actual-action@v0
        with:
          model: fable-5.1
```

Actual uses the model the input names only when your repository allows
it. Claude Fable 5.1 is allowed only where an owner turned it on in
Actual's settings, for every repository or per repository, because
Anthropic keeps its prompts and outputs for 30 days, and no file in the
repository can turn it on. When the repository does not
allow the model, or the value names no model Actual knows, the review uses
the repository's own model. The comment then says so in its first line
under the heading, as in "This review used Claude Opus 5.5, because this
repository does not allow Claude Fable 5.1." The check's summary says the
same. Without the input, the review uses the repository's model, and the
action asks for none. A run that brings its own key follows the same rule.

You may bring your own Anthropic API key instead, for example under a
data agreement of your own with Anthropic. Pass it as the
`anthropic-api-key` input:

```yaml
      - uses: nox-systems/actual-action@v0
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

With a key, the action calls Anthropic's API from the runner with that
key, and none of the review's model calls goes through Actual. Anthropic
bills those calls to your key, and Actual records the review as one that
brought its own key, which its billing prices apart. The key is then on
the runner, in the job's environment, though never inside the sandbox a
probe runs in. The model is still the one your org chose. The review
still finishes within the model budget per review your org sets, as
every review does.

When Anthropic refuses the key, or says its account cannot pay, the
review asks the model nothing more and posts what it found. An account
cannot pay when it has no credit, its billing failed, or it reached a
spend limit you set or its tier's monthly cap. Each region the review
did not check names the cause in the comment, and so does the `actual`
check's title, as in "Actual checked nothing because the repository's
own Anthropic account cannot pay for more calls". The check is neutral,
and the step then fails with one line that says the same, because only
you can replace the key or let its account pay (see "When Actual fails"
below).

### When a pull request is reviewed

Actual reviews a pull request once: the first push that is not a draft
gets the review. When that review checks nothing, such as when the base
tree does not build, the next push gets it. A later push runs your
workflow as before, while the action's step ends in seconds and exits 0.
The `actual` check run on that push concludes neutral, titled "Reviewed
at `abc1234`", with a Re-run button. A person with write access reviews
the new head by pressing it, by replying `/actual review` on the pull
request, or from the Reviews page of Actual's console. The action's job
then runs again. Such a follow-up review is granted the same model budget
as the first review.

The `review` input changes that for one step: `every-push` reviews every
push, and `once` reviews once whatever the repository's setting says.
Without it, the step follows the repository's setting, which an owner
changes on the repository's page in the console:

```yaml
      - uses: nox-systems/actual-action@v0
        with:
          review: every-push
```

### More than one review step

A repository can review parts of itself in separate jobs, such as a web
app and an API that build apart. Give each job's step a `name`, which
makes it a scope, and the `paths` it reviews, one glob pattern a line:

```yaml
  web:
    steps:
      - run: <the web app's build and tests>
      - uses: nox-systems/actual-action@v0
        with:
          name: web
          paths: |
            web/**
            packages/ui/**
  api:
    steps:
      - run: <the API's build and tests>
      - uses: nox-systems/actual-action@v0
        with:
          name: api
          paths: api/**
```

Each scope has its own check run, `actual / web` and `actual / api`
here, and its own model budget. It also has its own section of the pull
request's one comment, which only that scope's reviews replace, and the
comment holds the sections of at most 32 scopes. A step reviews only the
changed files its `paths` match, and a pull request that changes none of
them is skipped with a neutral check run, titled "No changes under this
scope's paths". A name holds at most 64 ASCII letters, digits, `.`, `-`,
and `_`, and case counts. The legs of a matrix job share their step's
scope, and so they are one review. One review of a head runs in a scope
at a time. A second step in the same scope on the same commit, such as
an unnamed step in another workflow file, ends at once. It learns that
the first step's review covers the commit.

### Steps that run in the background

GitHub can run a step in the background (`background: true`), and the job
then goes on to the next step while that step runs. Its failure shows only
at the next `wait` that names it, the next `wait-all`, or the end of the
`parallel` group that holds it. Before it reviews, the action reads the
job's workflow file and checks that one of those covers every background
step before its own. If one does not, the step's tests may still be
running or may already have failed. The action then prints one line that
names the step and exits 0 without reviewing. Put Actual's step after the
wait:

```yaml
      - id: tests
        run: <your test command>
        background: true
      - run: <your lint command>
      - wait: tests
      - uses: nox-systems/actual-action@v0
```

A `cancel` step ends the step it names, so a server that you start in the
background and cancel after your tests does not stop the review. A step
inside a `parallel` group runs beside the group's other steps, so put
Actual's step after the group. If the action cannot read or parse the
workflow file, it says why in one line and reviews anyway.

The action finds its own step as GitHub names it. Run through the action,
it is the step that uses the action, `nox-systems/actual-action` or a
fork's, at any version, a branch with a `/` in its name too. A copy in
the repository is `./<path>`, with or without a trailing `/`. The
runner's path to the action names its owner, repository, and version.
Run as a
binary in a `run` step, it is the step
`GITHUB_ACTION` names: the step's `id`, or `__run`, `__run_2`, and so on
for the job's `run` steps with no `id`. If it finds none, it prints
"actual: skipped the background-step check, because it could not find
its own step in the workflow." and reviews anyway.

### The job's timeout

A review has a time limit, from the moment it starts its run to its post:
30 minutes unless Actual sets another for your installation or your
repository. `budget.time_limit_ms` in `.actual/config.toml` can lower it
and never raise it. A fifth of the limit before it ends, the review starts no new work, then finishes and
posts what it found, with a neutral check whose title names the limit.

GitHub cancels a job that passes its `timeout-minutes`, and a cancelled
review posts nothing. Set the job's `timeout-minutes` above the time your
own steps take plus the limit. For example, a job whose install, build,
and tests take 12 minutes, reviewed under the 30-minute limit, needs more
than 42 minutes, and 50 leaves room for the action's own setup and the
post:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 50
```

### When Actual fails

The action fails open on Actual's own errors. When a review cannot run
or post because of Actual's side, the step exits 0 and shows one warning
annotation titled "Actual skipped this review", which says why. The
review then posts nothing, and the job stays green. Actual's side covers
its API when it cannot be reached, answers a server
error after the action's retries, or refuses a request the action built.
It also covers a release that does not download or verify, a bubblewrap
build that fails on a GitHub-hosted runner, a crash of Actual's binary,
and Actual's own pipeline. A call that fails in the middle of a review,
such as a write of a record, does not stop it, and the review still
posts.

The step fails, with one line that says why, only for a cause in your
own workflow or runner, because only you can fix it. The causes are the
following:

- The job has no `id-token: write` permission on a pull request from a
  branch of your own repository.
- The Actual GitHub App is not installed on the repository.
- An input or a variable the job sets is invalid, such as `paths`, or
  the `version` input names no release.
- The runner is not Linux on x86_64 or aarch64, a self-hosted runner
  lacks bubblewrap 0.11.0 or `/dev/fuse`, its curl is older than 7.71.0
  or lacks the trust roots to check GitHub's certificate, its git is
  older than 2.41.0, or no filesystem can hold the sandbox's layers. The
  review reads the diff with git's `--attr-source`, so the pull
  request's own `.gitattributes` cannot change how git prints it, and
  git added that option in 2.41.0.
- The checkout lacks the history the review reads, git fails on it, or
  the repository's `.actual/config.toml` or `.actual/recipe.toml` is
  invalid.

A job that has nothing to review prints one line and exits 0 with no
warning. That is a job of an event other than `pull_request`, a job of a
fork's pull request, a job Dependabot started that GitHub gave no token,
and a review that a newer push or the pull request's close ended.

## What the action does

The action is a composite of three steps:

1. It installs the probe sandbox. Probes run under bubblewrap 0.11.0 or
   later, whose overlays lay each tree out over the checkout's build
   outputs, and GitHub's `ubuntu-24.04` image ships 0.9.0. On a
   GitHub-hosted runner whose `bwrap` is missing or older than 0.11.0,
   the step builds 0.11.0 from its release tarball, checked against its
   pinned sha256, and installs it under `/usr/local`. It also lifts
   Ubuntu 24.04's AppArmor restriction on unprivileged user namespaces,
   which bubblewrap needs. A build that fails there skips the review with
   a warning (see "When Actual fails" above). A self-hosted runner keeps
   what a job changes, so there the step installs and changes nothing.
   It checks for bubblewrap 0.11.0 or later on `PATH`, for unprivileged
   user namespaces, and for a `/dev/fuse` the runner's user may open, and
   fails with one line naming what is missing.
2. It downloads the binary for the runner's architecture,
   `actual-action-x86_64-unknown-linux-musl.tar.xz` or
   `actual-action-aarch64-unknown-linux-musl.tar.xz`, from this
   repository's release that the `version` input names, or else the
   release that the action's `VERSION` file names. That file holds the
   release the commit you name the action by belongs to, so `@v0` and a
   pinned commit both run the matching binary. Each binary is linked
   statically against musl, so it runs whatever glibc the runner's image
   has. The step checks the download against the checksum the release
   publishes beside it, and the binary's version against the release's
   name, before it runs anything. A download that fails or does not
   verify skips the review with a warning (see "When Actual fails"
   above).
3. It runs the review in the checkout at `GITHUB_WORKSPACE`.

The review learns how your repository builds and runs one test file,
proves it with four handshake probes on each side, and keeps it, so
later reviews take it as stored. Each review then builds both sides and
writes and runs probes against the merge base and the head. It reads both trees from
the checkout's repository and lays the checkout's ignored directories
under each, so each side sees the dependencies and build outputs your
steps produced. It downloads no artifact. The comment's first line cites
the test reports and coverage your test step wrote, from the files the
Recipe's `reports` name, and says which files it read.

The action needs a Linux runner, x86_64 or aarch64, whose kernel allows
overlay mounts in a user namespace, which Linux does from 5.11 on, and a
`/dev/fuse` the runner's user may open. A probe writes the system's
directories through a FUSE layer that shows them as its own. GitHub's hosted runners have both. A runner whose bubblewrap is
older than 0.11.0 and that the first step could not update is refused with
a log line naming the version found, and a review on a runner with no
`/dev/fuse` abstains with "the runner has no /dev/fuse".

The review keeps the trees and the sandbox's layers under `RUNNER_TEMP`,
and the kernel refuses those layers on overlayfs. On a runner whose
`RUNNER_TEMP` is on overlayfs, such as one that runs in a container, the
action keeps them under `TMPDIR` or `/namespace/scratch` instead. When
neither can hold them, the action fails with one line before it starts a
review. On a Namespace runner, add `container.mount-scratch=true` to the
job's `runs-on`, which mounts a filesystem at `/namespace/scratch`.

## Inputs

The action reads the following inputs:

| Input | Default | What it is |
|---|---|---|
| `anthropic-api-key` | none | Your own Anthropic API key, which the review's model calls then use from the runner. Without one, they go through Actual's API, which calls the model with Actual's key. |
| `api-url` | `https://api.actual.ci` | Actual's API. |
| `model` | the repository's model | The model the review asks for (see "The model" above). |
| `name` | none | The scope the step reviews in, such as `web`. Without one, the step is the repository's one Actual step, and its check run is `actual`. |
| `paths` | every changed file | Glob patterns, one a line and at most 64, of the changed files the step reviews. |
| `review` | the repository's setting | `once` or `every-push` (see "When a pull request is reviewed" above). |
| `version` | the release in the action's `VERSION` file | The release whose binary runs, such as `v0.1.0`. |

## Tests in another job

A repository whose tests run in another job, or in several, can add the
review to a job of its own that waits for them. Name the job anything but
`actual`, because the App's check run has that name:

```yaml
  review:
    needs: [test]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      pull-requests: read
      checks: read
    steps:
      - uses: actions/checkout@<sha>
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: nox-systems/actual-action@v0
```

A job of its own starts from a fresh checkout with no dependencies and no
build outputs, and Actual installs nothing yet. The review there reports
every region as not checked, because the base did not build. Use the
one-step form above until the Recipe's `install` command runs in this form
in a later release. In this form no test of the pull request's own asserts
a claim the intent makes, because the steps before Actual's in the job ran
none of the repository's tests, and the review writes a probe for the
claim instead.
