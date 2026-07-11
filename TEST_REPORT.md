# Jupyter4NFDI automated notebook validation test report

**Test date:** 2026-07-11  
**Documentation tested:** <https://nfdi-jupyter.de/users/jupyterlab/ghactions/>  
**Test repository:** <https://github.com/arnim/test-nfdi-notebook-validation-20260711>  
**Fixture commit:** `ae8f83a`  
**Upstream action observed:** `NFDI-Jupyter/ghactions@742b80b1262cd527dcbf15092cb90c80bbe0d589`

No API token is stored in this repository or report.

## Result summary

The documented workflow works end-to-end. Directory filtering, nested-notebook discovery, successful execution, notebook-error propagation, GitHub failure annotations, secret masking, and job cleanup all behaved as expected.

| Test | Expected | Result |
|---|---|---|
| Documentation URL | HTTP 200 | PASS |
| Valid API token at `/hub/api/user` | HTTP 200 | PASS |
| Missing or invalid token | HTTP 403 | PASS |
| User token requesting admin-only user list | HTTP 403 | PASS (least privilege) |
| Workflow/YAML and notebook/JSON syntax | Valid | PASS |
| Selected-directory end-to-end workflow | Two selected notebooks pass; excluded failure is not run | PASS |
| All-notebooks end-to-end workflow | Intentional notebook error fails the GitHub job | PASS |
| Completed-job cleanup | No named servers remain | PASS |
| Secret hygiene | Exact API token absent from public logs | PASS |
| Action client unit/edge tests | Eight tests pass | PASS |

## Live end-to-end runs

### Selected directories — expected success

[GitHub Actions run 29133794107](https://github.com/arnim/test-nfdi-notebook-validation-20260711/actions/runs/29133794107) completed successfully in 56 seconds of job time.

The action was called with:

```yaml
notebook_dirs: '["notebooks", "examples"]'
```

Observed service result:

- `notebooks/pass.ipynb`: exit code 0
- `examples/nested/pass_nested.ipynb`: exit code 0
- Overall: exit code 0
- `excluded/intentional_failure.ipynb` was correctly excluded.

### All notebooks — expected failure

[GitHub Actions run 29133845539](https://github.com/arnim/test-nfdi-notebook-validation-20260711/actions/runs/29133845539) failed as expected in 32 seconds of job time.

Observed service result:

- Both passing notebooks: exit code 0
- `excluded/intentional_failure.ipynb`: exit code 1 with the expected `RuntimeError`
- Overall: exit code 1
- The composite action returned exit code 1, marked the workflow as failed, and emitted a GitHub error annotation.

The action's cleanup step ran after both the successful and failed executions. A subsequent API check showed zero named servers and zero active servers for the test account.

## Static and edge-case tests

The current `run_papermill.py` was compiled and tested locally with mocked HTTP responses. Eight tests covered:

1. JSON and comma-separated `notebook_dirs` parsing.
2. JSON and plain-text log parsing.
3. Correct Repo2Docker request payload construction.
4. Successful exit-code handling.
5. Per-notebook failure and nonzero exit-code propagation.
6. Plain-text exit-code fallback.
7. Unparseable-log fallback (exit code 117).
8. Malformed `user_options`, HTTP 400, and missing `Location` failures.

All eight passed.

## Findings and recommendations

### No release-blocking functional defect found

The main documented user path is functional and correctly distinguishes passing and failing notebooks.

### Reliability and maintainability

1. **Mutable action references.** The documentation recommends `NFDI-Jupyter/ghactions/...@main`, and the composite action uses `actions/setup-python@v6`. A later upstream change can alter existing workflows without a user commit. Publish versioned releases and recommend a major-version tag or immutable commit; pin third-party actions to full commit SHAs where practical.

2. **Over-broad trigger retry.** The client retries nearly every POST exception five times with 60-second delays, except HTTP 400. Permanent errors such as 401/403 therefore take roughly four minutes to fail. Fail immediately for 401/403/404/422 and retry only timeouts, connection errors, HTTP 429, and selected 5xx responses with bounded exponential backoff.

3. **Observed historical infrastructure failures.** Of the latest 20 upstream scheduled runs inspected (2026-02-16 through 2026-07-06), 17 succeeded and 3 failed. Two failures reported the JupyterHub limit of 15 named servers; one reported GitHub HTTP 429 while Repo2Docker resolved a ref. These were infrastructure/lifecycle failures rather than notebook failures. Monitoring, stale-server cleanup, and authenticated or cached GitHub ref resolution would improve reliability.

4. **Public identity in logs.** The action prints the complete job URL, which includes the JupyterHub username and job identifier. The API still requires authorization, but public-repository logs disclose this identifier. Consider logging an opaque URL or omitting the username portion.

5. **Input validation.** `notebook_dirs` accepts any JSON list, including non-string values. Validate that every entry is a nonempty relative string before sending the request.

6. **Minor log clarity.** Cleanup prints `Job cancelled via cleanup` even after a job has already stopped successfully. `Job cleaned up` would be less misleading.

7. **Permissive live API payload validation.** An authenticated `POST {}` to `/hub/api/job` returned HTTP 201 and allocated a job instead of rejecting the missing repository options. The test job was immediately deleted. If a default job is intentional, document it; otherwise require the Repo2Docker fields before allocating resources.

### Documentation corrections

- Correct `papaermill` to `papermill`.
- In the badge section, `_repotype_` appears to mean the repository name; `_reponame_` would be clearer.
- Explain that the workflow's GitHub **environment** name must match `environment: jupyter4nfdi`; users who prefer repository-level secrets can omit that environment setup and adjust the workflow accordingly.
- Consider recommending a tagged action release rather than `@main`.

## Cleanup status

The repository is intentionally manual-only: neither workflow has a schedule. It can be safely deleted after review.
