```json
{
    "[python]": {
        "editor.defaultFormatter": "ms-python.black-formatter",
    },
    "python.formatting.provider": "none",
    "python.envFile": "${workspaceFolder}/.env",
    "terminal.integrated.env.osx": {
        "PYTHONPATH": "${env:PYTHONPATH}:${workspaceFolder}"
    },
    "python.linting.enabled": false,
    "commentTranslate.source": "Google",
    "python.testing.pytestArgs": [
        "tests",
        "--no-header",
        "--no-summary",
        "--color=yes",
        "-s"
    ],
    "python.testing.unittestEnabled": false,
    "python.testing.pytestEnabled": true
}
```

> conda run -n pp_semantic_retrieval --no-capture-output python ~/.vscode/extensions/ms-python.python-2023.6.1/pythonFiles/get_output_via_markers.py ~/.vscode/extensions/ms-python.python-2023.6.1/pythonFiles/testing_tools/run_adapter.py discover pytest -- --rootdir . --cache-clear --no-header --no-summary --color=yes -s tests