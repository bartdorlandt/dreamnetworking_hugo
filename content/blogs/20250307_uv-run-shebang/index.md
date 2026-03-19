---
title: "Improved shebang for your python standalone script"
date: 2025-03-07
description: "Set up the shebang for your script using uv run"
image: "/blogs/uv_run_shebang/images/uv_shebang.png"
tags:
  - uv
  - python
  - script
---
Sometimes you just want a script, without having to specify a virtualenv or polluting your global python environment. Given [PEP-0723](https://peps.python.org/pep-0723/) we can build scripts as we would normally, though now add a few comments to it to specify the dependencies.

When we combine that with [uv](https://docs.astral.sh/uv/guides/scripts/) we can easily run the script without having to activate a virtualenv or specify the python interpreter. And it will just take care of it. Assume the script has the name `script.py` we would call it with `uv run -s script.py`.

This is great on its own, it still is less ideal when you want to be able to call this script as part of your path. How can we fix this?

Normally we would have a shebang in the python file pointing to a executable with a static path or using `env` to find the python interpreter.

But wouldn't it be great if we could have a shebang that would take care of it and will have the `uv run` as part of it and you would even have to think about it anymore when trying to run the command from the terminal?

## The fix

Let's update the shebang and make it automatically use `uv run` to run the script. This way we can just call the script as if it was a normal executable.

```bash
#!/usr/bin/env -S uv run --script
```

The `-S` for `env` has the following in the man page:
```
    -S string
            Split apart the given string into multiple strings, and process each of the resulting strings as separate arguments
            to the env utility.  The -S option recognizes some special character escape sequences and also supports
            environment-variable substitution, as described below.
```

and the `--script` for `uv` has the following:

```
  -s, --script
          Run the given path as a Python script.

          Using `--script` will attempt to parse the path as a PEP 723 script, irrespective of its extension.
```

## Example

```python
#!/usr/bin/env -S uv run --script

# /// script
# requires-python = ">=3.12"
# dependencies = [
#   "click",
# ]
# ///

import json
from ipaddress import ip_address, ip_network
from pathlib import Path

import click


@click.command()
@click.argument("peer_ip")
def main(peer_ip: str) -> None:
    JSONFILE = Path("data.json")
    json_data = json.loads(JSONFILE.read_text())
    ...


if __name__ == "__main__":
    main()

```

## Conclusion

With this in place, we can just call the script from anywhere on the CLI and have `uv` take care of the dependencies and we will not have to provide the `uv run --script` ourselves.

    /path/to/script.py

and its done.
