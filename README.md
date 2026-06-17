# Contribution #1: apiserver: return 400 instead of 500 for invalid DeleteOptions dryRun type

**Contribution Number:** 1  
**Student:** Quoc-Viet Bui  
**Issue:** [kubernetes/kubernetes#139490 — apiserver: return 400 instead of 500 for invalid DeleteOptions dryRun type](https://github.com/kubernetes/kubernetes/issues/139490)  
**PR:** [kubernetes/kubernetes#139767](https://github.com/kubernetes/kubernetes/pull/139767)  
**Working branch:** [fix/delete-options-dryrun-400](https://github.com/vietbui1999ru/kubernetes/tree/fix/delete-options-dryrun-400)  
**Status:** In-Progress

---

## Why I Chose This Issue

I chose this issue because it is straightforward and is within scope and depth of one code path in the API server request-handling layer (`delete.go` → error mapping in `responsewriters/status.go`). The root cause is traceable without too much experience in Go, and fixing it has direct visibility on API usability and debuggability. I also think it's a great introduction to learning how Kubernetes handles Go errors for HTTP status codes internally.

---

## Understanding the Issue

### Problem Description

When a DELETE request body contains `dryRun` as a JSON string (e.g. `"All"`) instead of a JSON array of strings (e.g. `["All"]`), the API server's JSON decoder returns a standard Go `json.UnmarshalTypeError`. That error is not wrapped as a `BadRequest` status error before it reaches the HTTP response writer, so the response writer falls back to a generic HTTP 500.

### Expected Behavior

The API server should return HTTP 400 Bad Request with a clear error message, since the client submitted an invalid request body (wrong JSON type for `dryRun`).

### Current Behavior

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "json: cannot unmarshal string into Go struct field DeleteOptions.dryRun of type []string",
  "code": 500
}
```

### Affected Components

- `staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go` — `DeleteOptions` struct defining `DryRun []string` (~line 600)
- `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/delete.go` — DELETE and DeleteCollection request body decoding (~lines 86–127, 263–302)
- `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/status.go` — maps errors to HTTP status codes; raw Go errors without a status code default to 500
- `staging/src/k8s.io/apimachinery/pkg/api/errors/errors.go` — `NewBadRequest()` helper that produces a properly coded 400 status error

---

## Reproduction Process

### Environment Setup

Forked `kubernetes/kubernetes` and cloned locally. Key setup notes:

- Go workspace project (`go.work`), multiple staging modules in `staging/src/k8s.io/`
- No dev container available; ran in bare Linux environment
- `gh` CLI required re-authentication (`gh auth login`)
- Go binary path needed manual configuration; used `make` targets instead
- Staging modules are source of truth for `k8s.io/*` packages — edits happen in `staging/src/`, then `make update` re-vendors

### Steps to Reproduce

1. Start a Kubernetes cluster (v1.33.1 confirmed affected).
2. Run `kubectl proxy --port=8001`.
3. Send:
   ```shell
   curl -X DELETE 'http://127.0.0.1:8001/api/v1/namespaces/default/configmaps' \
     -H 'Content-Type: application/json' \
     --data-raw '{"dryRun":"All"}'
   ```
4. Observe HTTP 500 with the unmarshal error message above.

### Reproduction Evidence

Code path traced in `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/delete.go`:

```
line 88:  body read via limitedReadBodyWithRecordMetric()
line 96:  input serializer negotiated via NegotiateInputSerializer()
line 104: obj, gvk, err := Decode(body, &defaultGVK, options)
          -> versioning.go:139 -> JSON unmarshal of dryRun fails
line 105: err = errors.NewBadRequest(err.Error())   // wraps as 400
line 107: scope.err(err, w, req)                    // writes response
```

The handler at `delete.go:104-108` already wraps decode errors as `errors.NewBadRequest()` (HTTP 400), meaning the code handles the type mismatch correctly. The gap was **missing test coverage** for the `dryRun` field specifically — existing tests only covered `orphanDependents` (*bool) and `apiVersion` (string) with invalid types.

---

## Solution Approach (Not yet reviewed and confirmed)

### Analysis

When `DeleteOptions.dryRun` (typed `[]string`) receives a JSON string `"All"`, the `kjson.UnmarshalCaseSensitivePreserveInts` call in `json.go:270` fails with a type mismatch. The error propagates via `versioning.Decode()` → `delete.go:104`, where it's wrapped as `errors.NewBadRequest(err.Error())` yielding HTTP 400. The existing tests verified this for `orphanDependents` (*bool → string mismatch) and `apiVersion` (string → bool mismatch) but had no coverage for `dryRun` ([]string → string mismatch).

### Proposed Solution

The same pattern exists in `create.go` and `update.go` which use `transformDecodeError()` for richer error messages. The delete tests already had 4 test cases for invalid JSON types; adding a 5th for dryRun follows the exact same pattern.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** When `DeleteOptions.dryRun` (typed `[]string`) receives a JSON string `"All"`, the `kjson.UnmarshalCaseSensitivePreserveInts` call in `json.go:270` fails with a type mismatch. The error propagates via `versioning.Decode()` → `delete.go:104`, where it's wrapped as `errors.NewBadRequest(err.Error())` yielding HTTP 400. The existing tests verified this for `orphanDependents` (*bool → string mismatch) and `apiVersion` (string → bool mismatch) but had no coverage for `dryRun` ([]string → string mismatch).

**Match:** The same pattern exists in `create.go` and `update.go` which use `transformDecodeError()` for richer error messages. The delete tests already had 4 test cases for invalid JSON types; adding a 5th for dryRun follows the exact same pattern.

**Plan:**
1. Add test case `"dryRun invalid Go type"` with body `{"dryRun": "All"}` expecting HTTP 400 to `TestDeleteResourceDeleteOptions` in `delete_test.go`
2. Add identical test case to `TestDeleteCollectionDeleteOptions`
3. Run the test suite to verify

**Local Implementation only, link is a dead PR:** See [PR #139767](https://github.com/kubernetes/kubernetes/pull/139767)

**Files changed:**
- `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/delete_test.go` — added 2 test cases

**Review:** Follows Kubernetes contribution guidelines: one logical change, regression test added, no `@mentions` or `Fixes #` in commit message, `/kind bug` label on PR.

**Evaluate:**
- `TestDeleteResourceDeleteOptions/dryRun_invalid_Go_type` — verifies 400 returned for single resource DELETE
- `TestDeleteCollectionDeleteOptions/dryRun_invalid_Go_type` — verifies 400 returned for collection DELETE
- Run `make test WHAT=./staging/src/k8s.io/apiserver/pkg/endpoints/handlers` to confirm all existing tests still pass

---

## Testing Strategy

### Unit Tests

- [x] `TestDeleteResourceDeleteOptions/dryRun_invalid_Go_type` — verifies 400 returned for single resource DELETE with invalid dryRun type
- [x] `TestDeleteCollectionDeleteOptions/dryRun_invalid_Go_type` — verifies 400 returned for collection DELETE with invalid dryRun type

### Integration Tests

- [x] Run `make test WHAT=./staging/src/k8s.io/apiserver/pkg/endpoints/handlers`

### Manual Testing

Verified via `curl` request to the API server proxy endpoint before and after the fix.

---

## Implementation Notes

### Week 1 Progress

Selected issue, explored codebase to identify affected files and code path, set up README, traced the full code path from handler to error response, confirmed the handler already wraps decode errors as 400, identified that only test coverage was missing.

### Code Changes

- **Files modified:** `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/delete_test.go` — added 2 test cases
- **Key commits:** [PR #139767](https://github.com/kubernetes/kubernetes/pull/139767)
- **Approach decisions:** Added regression test coverage for `dryRun` field type mismatch in both `TestDeleteResourceDeleteOptions` and `TestDeleteCollectionDeleteOptions`, following the existing pattern for `orphanDependents` and `apiVersion`.

---

## Pull Request

**PR Link:** [kubernetes/kubernetes#139767](https://github.com/kubernetes/kubernetes/pull/139767)

**PR Description:** Adds regression test coverage for invalid `dryRun` JSON type in DELETE operations. When `DeleteOptions.dryRun` receives a string `"All"` instead of `["All"]`, the handler already wraps the decode error as `errors.NewBadRequest()` (HTTP 400) — this PR adds test cases to verify the behavior.

**Maintainer Feedback:**
- Pending PR and waiting for Reviewers

**Status:** not submitted, plan and local reviewing

---

## Learnings & Reflections

### Technical Skills Gained

*To be filled in at completion.*

### Challenges Overcome

*To be filled in at completion.*

### What I'd Do Differently Next Time

*To be filled in at completion.*

---

## Resources Used

- [Issue #139490](https://github.com/kubernetes/kubernetes/issues/139490)
- [Kubernetes CONTRIBUTING.md](https://github.com/kubernetes/kubernetes/blob/master/CONTRIBUTING.md)
- `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/delete.go`
- `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/status.go`
- `staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go`
- `staging/src/k8s.io/apimachinery/pkg/api/errors/errors.go`
