# Run Records

A run is one concrete execution of an experiment.

Each experiment may have multiple runs if it is repeated with different models, providers, configuration, versions, or environments.

## Minimum Observability

Each run should record:

- Run ID.
- Date.
- System.
- System version or commit when known.
- Operating environment.
- Relevant configuration.
- Requested model.
- Observed model when observable.
- Provider.
- Gateway if used.
- Prompt or task.
- Commands executed.
- Relevant logs.
- Observed result.
- Available metrics.
- Errors or retries.
- Limitations.

Use `templates/run.md` when creating a new run record.
