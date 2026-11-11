# set the `PYTHONPATH` when debugging with **F5** in Visual Studio Code (VS Code)

## Option 1: Use `env` in `launch.json`

When you press **F5**, VS Code uses the configuration in `.vscode/launch.json`.

You can set `PYTHONPATH` (and other environment variables) directly there:

``` json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Debug App",
      "type": "python",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal",
      "env": {
        "PYTHONPATH": "${workspaceFolder}/src"
      }
    }
  ]
}
```

👉 Replace `${workspaceFolder}/src` with the path you want added to the Python module search path.

When you press **F5**, VS Code will:

- Set `PYTHONPATH` before launching the debugger.

- Your app will see the correct import paths.

------------------------------------------------------------------------

## Option 2: Use `.env` file

You can also define environment variables (including `PYTHONPATH`) in a `.env` file, and tell VS Code to use it.

1.  Create a file called `.env` in your project root:

    ``` bash
    PYTHONPATH=src
    ```

2.  In `launch.json`, reference it:

    ``` json
    {
      "name": "Python: Debug App",
      "type": "python",
      "request": "launch",
      "program": "${file}",
      "envFile": "${workspaceFolder}/.env"
    }
    ```

Then pressing **F5** will load `PYTHONPATH` from `.env` before debugging starts.

------------------------------------------------------------------------

## Option 3: Add path dynamically in code (less preferred)

If you can’t modify `launch.json`, you can add the path programmatically:

``` python
import sys
sys.path.append("/path/to/your/src")
```

…but this is usually not ideal — better to configure it in VS Code.
