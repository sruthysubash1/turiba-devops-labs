# Lab 6 · Test what you ship, ship a release

GitHub user: sruthysubash1 · Reference pipeline used (Part 0 fallback): no

## 1 · Smoke test

- The names of my smoke-test steps in the `image` job: `Build image for smoke test`, `Start container for smoke test`, `Smoke test GET /`, `Show container logs on smoke failure`, and `Stop smoke test container`.
- What my smoke test asks, and what makes it fail: It builds the Docker image, starts a container and sends an HTTP GET request to `/` on port 5000. It fails if the container does not start correctly or the API does not return a successful HTTP response. On failure, it displays the container logs.
- Part 1, before my first run: I predicted that my code would pass the smoke test because the API should start and respond to GET /. What happened: The smoke test passed on the main branch after I merged the workflow.
- Part 2, before the checks finished: I predicted that the `image` check would go red because the Docker image might be missing a file needed by the application. What happened: The image check failed at the smoke test, while `lint`, `unit` and `integration` stayed green.

## 2 · What broke and how I found it

- The check and the step that went red: The `ci / image` check failed in the `Smoke test GET /` step of PR #8.

- The log line that showed the cause (copied from the log):

```text
Error: Cannot find module './banner'
Require stack:
- /app/server.js
```

- The cause, in my own words: The new startup-banner feature required `banner.js`. The application source code contained this file, but `api/.dockerignore` excluded it from the Docker build context. As a result, the file was missing inside the built image, and the Node.js application could not start.

- Why `lint`, `unit` and `integration` stayed green: These checks tested the source code in the repository checkout. They did not test whether every required file was included in the final Docker image. The missing file was only discovered when the smoke test started the container.

- My fix, and the commit that made it: I removed the `banner.js` exclusion from `api/.dockerignore`, keeping the other ignore rules. The fix was committed as `c888f6d` with the message `fix(api): include startup banner in Docker image`. After the fix, all four checks passed and PR #8 was merged.

## 3 · Release

- Release page: https://github.com/sruthysubash1/turiba-devops-labs/releases/tag/v0.2.0
- Image: `ghcr.io/sruthysubash1/turiba-devops-labs/api:0.2.0`, platforms (from `docker buildx imagetools inspect`): `linux/amd64` and `linux/arm64`.
- One thing in Part 3 that didn't work the first time, and how I fixed it: Docker was initially unavailable in my Ubuntu WSL terminal because Docker Desktop WSL integration was not enabled. I enabled integration for Ubuntu in Docker Desktop settings, clicked Apply & Restart, and reopened Ubuntu. Then Docker commands worked and I verified both image architectures.
