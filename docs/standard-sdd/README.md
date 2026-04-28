# Standard SDD Profile

Standard SDD projects live under `projects/{project-id}/`.

Each project owns its own full SDD chain:

```text
projects/{project-id}/
├── 01-requirements/
├── 02-user-stories/
├── 03-spec/
├── 04-architecture/
├── 05-design/
│   └── contracts/
└── 06-tasks/
```

Graph-participating documents should set:

```yaml
profile: "standard-sdd"
project_id: "{project-id}"
```

The current baseline project is `control-tower`.
