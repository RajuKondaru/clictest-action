# ClicTest Action (GitHub Actions + Gitea Actions)

Runs ClicTest test suites from a workflow, the same way the ClicTest Jenkins plugin does:
authenticate (`/getuser`), trigger one execution per test suite x browser (`/execute`),
poll `/executionStatus` until `COMPLETED` / `FAILED` / `TERMINATED`, then fail the job if any execution failed.

No build step: `ClicTest.java` runs through the JDK source launcher (`java ClicTest.java`), so the runner only needs a JDK (installed by `actions/setup-java`).

## Prerequisites

- The ClicTest service URL must be reachable from the runner. GitHub-hosted runners need a public endpoint.
- Repository secrets `CLICTEST_USER` and `CLICTEST_PASSWORD`.
- Local-browser runs (`client-ip`) need a self-hosted runner on a machine with the ClicTest agent.
- Gitea: Actions enabled (`[actions] ENABLED=true` in `app.ini`) and an `act_runner` registered. The runner image needs `bash`. The default `node` images have it, but not Java, so keep `java-version` set.

## Usage

GitHub:

```yaml
name: ClicTest
on: [push, workflow_dispatch]

jobs:
  clictest:
    runs-on: ubuntu-latest
    steps:
      - id: clictest
        uses: RajuKondaru/clictest-action@v1
        with:
          env-name: preprodone
          project-name: Sugarcrm
          username: ${{ secrets.CLICTEST_USER }}
          password: ${{ secrets.CLICTEST_PASSWORD }}
          test-suites: |
            [
              {"testSuiteName": "Wallmart_Test", "testSuiteVersion": "1",
               "url": "https://example.com", "platform": "windows10",
               "browsers": ["chrome", "firefox"]}
            ]
      - run: echo "Report ${{ steps.clictest.outputs.html-report-url }}"
        if: always()
```

Gitea: same workflow under `.gitea/workflows/` (or `.github/workflows/`), but reference the action by full URL, because a bare `owner/repo` resolves against github.com by default:

```yaml
        uses: http://gitea.smartintelectual.com:3000/smartintelectual/clictest-action@v1
```


## Configurator UI (GitHub Pages)

`docs/index.html` is a single-page configurator, the GitHub counterpart of the ClicTest Azure DevOps hub:
cascading dropdowns (environment, project, test suite, version, platform, URL, browsers), multiple test suites,
then **Copy** the `test-suites` JSON / workflow step, or **Run now** to start the `ClicTest Run` workflow.

Setup:

1. Repository settings > Pages > Deploy from branch `master`, folder `/docs`. The page is served at
   `https://rajukondaru.github.io/clictest-action/`. Gitea has no Pages: host `docs/index.html` on any static web server.
2. In each repository that runs tests, add `examples/clictest-run.yml` as `.github/workflows/clictest-run.yml`
   plus the `CLICTEST_USER` / `CLICTEST_PASSWORD` secrets.
3. Run now needs an access token with Actions write permission on that repository
   (GitHub fine-grained token: "Actions: Read and write"; Gitea: token with `write:repository`).
   The token stays in the browser tab and goes only to the API base you enter.

ClicTest server requirements for the page:

- **HTTPS.** The page is served over HTTPS, so browsers block calls to an `http://` service URL (mixed content).
- **CORS.** The service must answer with `Access-Control-Allow-Origin` for the page's origin
  (e.g. `https://rajukondaru.github.io`).

Lookup calls used: `/envdetails`, `/projectdetails`, `/testsuitedetails`, `/configurationdetails`,
`/testsuiteversion`, `/browserversion` (same as the Jenkins plugin's dropdowns).

## Inputs

| Input | Required | Default | Notes |
|---|---|---|---|
| `env-name` | yes | | ClicTest environment |
| `project-name` | yes | | ClicTest project |
| `username` | yes | | use a secret |
| `password` | yes | | use a secret |
| `test-suites` | yes | | JSON array: `testSuiteName`, `testSuiteVersion`, `url`, `platform`, `browsers[]` |
| `service-url` | no | preprod service URL | same as `serviceurl` in the plugin's `config.properties` |
| `job-name` | no | `GITHUB_REPOSITORY` | sent as `jobName`; build number is `GITHUB_RUN_NUMBER` |
| `client-ip` | no | empty | local ClicTest agent IP |
| `max-parallel` | no | `10` | |
| `poll-interval-seconds` | no | `10` | |
| `timeout-minutes` | no | `60` | per execution |
| `java-version` | no | `17` | `''` skips setup-java |

## Outputs

`result` (`success` / `failure`), `html-report-url`, `pdf-report-url`. A results table is also written to the job summary.

## Local check

```bash
java ClicTest.java --self-test
```

## Security note

`/getuser` takes the password as a query parameter, so it can show up in ClicTest server and proxy access logs. Use HTTPS for `service-url`, and move the server to POST or token auth when possible.

## Releasing

`action.yml` sits at the repository root, so workflows reference `owner/clictest-action@<tag>`.
Tag a release and move the major tag:

```bash
git tag v1.0.0 && git tag -f v1 && git push origin v1.0.0 && git push -f origin v1
```

Related project: ClicTest Jenkins plugin (same REST flow, see `ClictestCIPublisher`).
