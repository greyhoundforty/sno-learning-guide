# Lab step schema

Each lab is a YAML list of steps. The guide renders them and the verifier checks them. See [`docs/architecture.md`](../docs/architecture.md) for the flow and RBAC.

## Fields

| Field | Type | Required | Description | Example |
| --- | --- | --- | --- | --- |
| `id` | string (slug) | yes | Unique within the lab. Lowercase letters, digits, hyphens. Used in `POST /api/verify/<id>` and as the SVG node key. | `create-project` |
| `title` | string | yes | Short heading shown on the step card. | `Create the demo project` |
| `instructions` | markdown string | yes | What the learner does. Commands go in fenced code blocks. | `Run \`oc new-project demo\`.` |
| `check_type` | enum | yes | Which check the verifier runs. One of the values below. | `project_exists` |
| `expected` | map | yes | Parameters for the check; shape depends on `check_type`. Use `{}` for `manual`. | `{ name: demo }` |
| `hint` | string | no | Text shown on failure when the check gives no specific message. | `Did you run oc new-project?` |

The verifier rejects a step whose `expected` map does not match the shape for its `check_type`. Unknown keys are errors.

## Check types

| `check_type` | `expected` keys | Passes when |
| --- | --- | --- |
| `project_exists` | `name` (string) | The project exists |
| `resource_exists` | `kind` (string, allow-listed), `name` (string), `namespace` (string) | The named resource exists |
| `deployment_ready` | `name` (string), `namespace` (string), `replicas` (int, optional) | Ready replicas equal desired, or `replicas` if given |
| `manual` | none | Always resolved in the guide; the API is not called |

`resource_exists.kind` must be one of the kinds granted in the verifier ClusterRole.

## Response

```json
{ "pass": false, "hint": "Project demo was not found." }
```

## Example

```yaml
- id: deploy-hello
  title: Deploy the hello app
  instructions: |
    Create a deployment in the `demo` project:

    ```
    oc create deployment hello --image=quay.io/example/hello -n demo
    ```

    Wait until the pod is running.
  check_type: deployment_ready
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
```
