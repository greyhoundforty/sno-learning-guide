# Lab step schema

A lab is one YAML file: a lab-level wrapper holding an ordered list of steps. The guide renders it and the verifier checks it. See [`docs/architecture.md`](../docs/architecture.md) for the flow, states, and RBAC.

## Lab fields

| Field | Type | Required | Description | Example |
| --- | --- | --- | --- | --- |
| `id` | string (slug) | yes | Lab id. Used in the route and the `localStorage` key. | `hello-sno` |
| `title` | string | yes | Lab heading. | `Hello on SNO` |
| `steps` | list of step | yes | Rendered and checked in list order. | see Example |

## Step fields

| Field | Type | Required | Description | Example |
| --- | --- | --- | --- | --- |
| `id` | string (slug) | yes | Unique within the lab. Lowercase letters, digits, hyphens. Used in `POST /api/verify/<lab-id>/<step-id>`. | `create-project` |
| `title` | string | yes | Short heading shown on the step card. | `Create the demo project` |
| `instructions` | markdown string | yes | What the learner does. Commands go in fenced code blocks. | ``Run `oc new-project demo`.`` |
| `check_type` | enum | yes | Which check the verifier runs. One of the values below. | `project_exists` |
| `expected` | map | yes | Parameters for the check; shape depends on `check_type`. Use `{}` for `manual`. | `{ name: demo }` |
| `requires` | list of step id | no | Steps that must be `pass` first. Defaults to the previous step; `[]` means none. | `[create-project]` |
| `diagram_node` | string | no | Value of the SVG `data-step` attribute to update. Defaults to `id`. | `app` |
| `hint` | string | no | Shown on `fail` when the response has no hint. | `Did you run oc new-project?` |
| `cleanup` | markdown string | no | Text telling the learner how to undo the step. Never executed. | ``Run `oc delete project demo`.`` |

Rules:

- Order is list order. Each `requires` id must exist earlier in the same lab; the verifier and lint reject dangling or forward references.
- The verifier rejects a step whose `expected` map does not match its `check_type`. Unknown keys are errors.

## Check types

| `check_type` | `expected` keys | Passes when |
| --- | --- | --- |
| `project_exists` | `name` (string) | The project exists |
| `resource_exists` | `kind` (enum below), `name` (string), `namespace` (string, omit for `Project` and `Namespace`) | The named resource exists |
| `deployment_ready` | `name` (string), `namespace` (string), `replicas` (int, optional) | Ready replicas equal desired, or `replicas` if given |
| `manual` | none | Resolved in the guide; the API is not called |

### resource_exists kinds

`kind` must be exactly one of these case-sensitive strings. Anything else is rejected with `invalid_expected` before any cluster call.

| Allowed `kind` |
| --- |
| `Project` |
| `Namespace` |
| `Pod` |
| `Service` |
| `ConfigMap` |
| `Deployment` |
| `Route` |

## Response

`status` is `pass`, `fail`, or `error`. A check that ran returns HTTP 200 with `pass` or `fail`. Otherwise `status` is `error` with an `error_kind` and the HTTP code listed in the architecture document.

```json
{ "status": "fail", "hint": "Project demo was not found." }
```

```json
{ "status": "error", "error_kind": "cluster_unreachable", "hint": "Cannot reach the cluster." }
```

On `pass`, `hint` is `""`.

## Example

````yaml
id: hello-sno
title: Hello on SNO
steps:
  - id: create-project
    title: Create the demo project
    instructions: Run `oc new-project demo`.
    check_type: project_exists
    expected:
      name: demo
    cleanup: Run `oc delete project demo`.

  - id: deploy-hello
    title: Deploy the hello app
    instructions: |
      Create a deployment in the `demo` project:

      ```
      oc create deployment hello --image=quay.io/openshifttest/hello-openshift:openshift -n demo
      ```

      Wait until the pod is running.
    check_type: deployment_ready
    requires: [create-project]
    diagram_node: app
    expected:
      name: hello
      namespace: demo
      replicas: 1
    hint: Check the pod status with `oc get pods -n demo`.

  - id: compare-notes
    title: Compare output with your notes
    instructions: Compare the command output with what you wrote down earlier.
    check_type: manual
    expected: {}
````
