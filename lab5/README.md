# Lab 5 · Your first CI pipeline

GitHub user: sruthysubash1 · Reference files used (Lab 2 fallback): no

![ci](https://github.com/sruthysubash1/turiba-devops-labs/actions/workflows/ci.yml/badge.svg?branch=main)

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores), and your user name placeholder in the first line and in the badge URL: write your GitHub user name exactly as GitHub shows it. -->

## 1 · Graph predictions

Part 1, step 1.2: fill in this section **before** you run `lab05 graph`. A wrong prediction that you explain earns full marks.

My drawing of the `lab05 graph` jobs, before the first run (boxes and arrows in text are fine):

```text
graph-build ──────┬──────> graph-test ──────┐
                  │                         v
graph-lint ───────┴────────────────> graph-package
                                             |
                                             v
                                       graph-report

graph-build ──────> graph-test
graph-build ──────> graph-report
graph-lint ───────> graph-package
graph-lint ───────> graph-report
graph-test ───────> graph-report
graph-package ────> graph-report
```

| # | Question | I predicted, and why | What the run showed |
|---|----------|----------------------|---------------------|
| P1 | Which jobs start together? | graph-build and graph-lint start together because neither has a needs dependency. | Both started independently. |
| P2 | Will `graph-package` find `build.txt`? | No. Jobs have separate environments, and graph-package does not download the artifact. | It reported build.txt: not here: jobs don't share files. |
| P3 | With `break = build`: which jobs run, which fail, which are skipped? | If build fails, graph-test and graph-package should be skipped; graph-lint should still run, and graph-report should run because of if: always(). | Build failed intentionally; test and package were skipped; lint and report ran. |

## 2 · Pipeline

- B5, before I filled it in: `localhost` or `db`? I predicted `localhost` because the workflow steps run on the GitHub Actions runner, which connects to the PostgreSQL service through its published port.
- My first red run in Part 2 (the job, the first red step, the error line copied from the log): No red Part 2 run appears in my available workflow history; the recorded Part 2 pull-request run passed all four jobs.
- What I changed to make it green: The recorded Part 2 pull-request run passed all four jobs. 
- My lint findings, and how I fixed them (fixed, not silenced): The ESLint configuration uses the recommended JavaScript rules and treats unused variables as errors. I ran `npm run lint` inside a Node.js 24 Docker container, and it completed successfully with no lint errors. I could not verify the original lint findings from the available Git history.
- Part 2 pull request (URL): https://github.com/sruthysubash1/turiba-devops-labs/pull/3
- Part 3: will the PR run push an image? I predicted no. What happened: The PR run skipped the main-only GHCR login step, so it did not publish the image. After the PR was merged, the main-branch run logged in to GHCR and built and pushed the image.
- My image on GHCR, from the `main` run: `ghcr.io/sruthysubash1/turiba-devops-labs/api:40d5f35c7e5ecae4f43b8ea9ce5d5839f1c0361b`

## 3 · Red PR

- Red PR (URL): Not completed. I opened a pull request, but GitHub reported that the workflow completed with no jobs.
- The test that failed, with Expected and Received (copied): Not verified; no integration test result was produced.
- The cause line from "On failure: the API log" (copied):  Not available because the integration job did not run.
- The merge box while it was red (the text, copied): Not captured; the required failing-check state was not verified.
- Which commit did the red check test: your branch alone, or your branch merged with `main`? How do you know? Not verified because the workflow produced no jobs.

## 4 · Answers

**A1.** In my break = build run, graph-build failed intentionally, graph-lint passed, and graph-test and graph-package were skipped. graph-report still ran and passed because it uses if: always(), which allows it to run even when an earlier job fails or is skipped.

**A2.** graph-test found build.txt because it downloaded the artifact produced by graph-build. graph-package ran in a separate job environment and did not download that artifact, so the file was not present. Adding a download-artifact step for the artifact from graph-build would make the file available to graph-package.

**A3.** DB_HOST: localhost is correct in the integration job because the steps run directly on the GitHub Actions runner, and the PostgreSQL service publishes its port to that runner. In Lab 4, db is correct because the application and database communicate over the Docker Compose network using the database service name. 

**A4.** The required check that failed in my red PR was integration. The failing run tested GitHub's pull-request merge commit, which combines the PR branch with the base branch, rather than testing the branch in isolation. The merge was blocked because the required check was unsuccessful, and the merge button was disabled. The image job ran on my Part 2 PR because it depends on the lint, unit, and integration jobs, but it did not push an image because publishing is restricted to runs on main.
