# Contribution #1: apiserver: return 400 instead of 500 for invalid DeleteOptions dryRun type

**Contribution Number:** 1  
**Student:** Quoc-Viet Bui  
**Issue:** [kubernetes/kubernetes#139490 — apiserver: return 400 instead of 500 for invalid DeleteOptions dryRun type](https://github.com/kubernetes/kubernetes/issues/139490)  
**Status:** Phase I — In Progress

---

## Why I Chose This Issue

The Kubernetes API server returns HTTP 500 (Internal Server Error) when a client sends a DELETE request with `dryRun` as a plain string instead of the required `[]string` array — a clear client mistake that should produce a 400 (Bad Request). This matters because a 500 signals a server-side fault and misleads operators and monitoring systems into thinking the cluster is unhealthy when the real problem is simply a malformed request body.

I chose this issue because it is tightly scoped to one code path in the API server request-handling layer (`delete.go` → error mapping in `responsewriters/status.go`), the root cause is traceable without deep Go expertise, and fixing it has direct, visible impact on API usability and debuggability. It is also a great way to learn how Kubernetes maps Go errors to HTTP status codes internally — a pattern that appears throughout the apiserver.

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

*To be filled in during Phase II.*

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

- **Commit showing reproduction:** *To be filled in during Phase II*
- **Screenshots/logs:** *To be filled in during Phase II*
- **My findings:** *To be filled in during Phase II*

---

## Solution Approach

### Analysis

*To be filled in during Phase II.*

### Proposed Solution

*To be filled in during Phase II.*

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** *To be filled in during Phase II*

**Match:** *To be filled in during Phase II*

**Plan:** *To be filled in during Phase II*

**Implement:** *To be filled in during Phase II*

**Review:** *To be filled in during Phase II*

**Evaluate:** *To be filled in during Phase II*

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: *To be filled in during Phase III*
- [ ] Test case 2: *To be filled in during Phase III*
- [ ] Test case 3: *To be filled in during Phase III*

### Integration Tests

- [ ] *To be filled in during Phase III*

### Manual Testing

*To be filled in during Phase III.*

---

## Implementation Notes

### Week 1 Progress

Selected issue, explored codebase to identify affected files and code path, set up README.

### Code Changes

- **Files modified:** None yet
- **Key commits:** *To be filled in during Phase III*
- **Approach decisions:** *To be filled in during Phase III*

---

## Pull Request

**PR Link:** *To be filled in during Phase IV*

**PR Description:** *To be filled in during Phase IV*

**Maintainer Feedback:**
- *To be filled in during Phase IV*

**Status:** Not yet submitted

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
