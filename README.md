# Jupyter4NFDI notebook-validation test repository

Disposable end-to-end test repository for the Jupyter4NFDI automated notebook validation GitHub Action.

See the [test report](TEST_REPORT.md) for results, run links, and recommendations.

## Test layout

- `notebooks/pass.ipynb`: should execute successfully.
- `examples/nested/pass_nested.ipynb`: should execute successfully.
- `excluded/intentional_failure.ipynb`: deliberately raises an exception.
- `validate-selected.yml`: validates only `notebooks` and `examples`; expected to pass.
- `validate-all-expected-failure.yml`: validates every notebook; expected to fail.

The workflows are manual-only so this repository consumes no scheduled resources.
