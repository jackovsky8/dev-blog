---
layout: post
title:  Designing An Universal Test Tool!
date:   2024-03-23 14:37:17 +0100
categories: [ universal-test-tool, python, development ]
tags: [ universal-test-tool, python, development ]
---

The problem I try to solve here, is to run integration and system tests for a lot of different type of services which either previously have been made manually, with different tools or with copy and paste scripts.

The idea first started with another scripts which handle different task by a configuration file, so the script could stay the same for all tests and only the configuration had to be rewritten.

This approach really soon came to a dead end, since the script was not readable due to its complex structure and also
not really testable. So every little change on one end broke another end.

So the whole script had to be designed from scratch to be more robust and still keep the flexibility which is required.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

# The design

{: .mt-4 .mb-0 }

<!-- markdownlint-restore -->

The design approach was to have a base script which builds a kind of framework and starts the actual tests in plugins.
With this approach the base script does not need to know about the tests.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

## Requirements

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

1. A CLI to start the program
2. The tests should be easy configurable
3. The software should be easy testable and stable

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

## Designing The CLI

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

It is quite easy to create a cli with the element from the python standard library [argparse](https://docs.python.org/3/library/argparse.html).
So we only declare what we need, add a text for printing the help and if required some default values.

So the easiest way to start the cli should be, that we access the folder where the test configuration is stored and run
**_test-tool_**.
If the naming of the configuration files is right, the no further arguments are needed.

As to see in the code below, there are to different config files per test. The **_Configuration-File_** `config.yaml`
and the **_Data-File_** `data.yaml`. The **_Configuration-File_** contains the test steps which we want to run. The *
*_Data-File_** contains complementary data. The ides behind this separation is, that if you want to run the tests in
multiple environments you just add another **_Data-File_** and there is no need to change the test steps itself.

The argument **_project_** makes it possible to start a test from everywhere. The path given here will be used as **_cwd_**. So that we can use relative urls in the test configuration.

The other arguments are pretty self-explanatory, so we won't discuss them here further.

The finished file looks like:

```python
"""
CLI interface for test_tool project.

It is executed when the program is called from the command line.
"""
from argparse import ArgumentParser
from logging import INFO, DEBUG, basicConfig
from os import getcwd

import pkg_resources
from test_tool.base import run_tests


def main() -> None:  # pragma: no cover
    """
    This is the program's entry point.
    """
    parser = ArgumentParser(
        prog="test-tool",
        description="This programm runs tests configured in a yaml file.",
        epilog="universal-test-tool Copyright (C) 2023 jackovsky8",
    )

    parser.add_argument(
        "-p",
        "--project",
        action="store",
        help="The path to the project.",
        default=getcwd(),
    )

    parser.add_argument(
        "-ca",
        "--calls",
        action="store",
        help="The filename of the calls configuration.",
        default="calls.yaml",
    )

    parser.add_argument(
        "-d",
        "--data",
        action="store",
        help="The filename of the data configuration.",
        default="data.yaml",
    )

    parser.add_argument(
        "-c",
        "--continue_tests",
        action="store_true",
        help="Continue the tests even if one fails.",
        default=False,
    )

    parser.add_argument(
        "-o",
        "--output",
        action="store",
        help="Create a folder with the logs of the tests.",
        default="runs/%Y%m%d_%H%M%S",
    )

    version: str = pkg_resources.require("universal_test_tool")[0].version
    parser.add_argument(
        "-v",
        "--version",
        action="version",
        version=f"%(prog)s {version}",
        help="Show the version of the program.",
    )

    parser.add_argument(
        "-X",
        "--debug",
        action="store_true",
        help="Activate debugging."
    )

    # Parse the arguments
    args = parser.parse_args()

    log_format: str = "%(asctime)s | %(levelname)s | %(name)s | %(message)s"
    log_level = INFO
    if args.debug:
        log_level = DEBUG
    basicConfig(level=log_level, format=log_format)

    run_tests(
        args.project, args.calls, args.data, args.continue_tests, args.output
    )


if __name__ == "__main__":  # pragma: no cover
    main()
```

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

## Configuring The Test

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

Since the requirement for this tool was that the tests are easy configurable, the decision for yaml[^yaml] was made.
Yaml is easy read and writeable for humans and is powerful enough to design complex test cases.

The basic idea was we want to run one run the test steps one after another. Sometimes it might be faster to run tests in
parallel, but since this program is designed for complex test cases which not necessarily are idempotent we want to keep
the order.

So basically the **_Configuration-File_** contains a list of test steps. A test step contains the **_type_** and the *
*_call_**. The **_type_** defines the plugin with which the test step will be executed. And the **_call_** contains
detailed information for the execution. Even if you will see[^plugins], that we take an effort to keep the structure for
every call similar, it might look quite different for every plugin.

So basically the config file will look like:

```yaml
- type: FOO
  call:
    __data__: for_test_step_1
- type: BAR
  call:
    __data__: for_test_step_1
```

The **_Data-File_** contains, as said above complementary data. The structure of this one is not defined. It only needs to fit to the test steps.

```yaml
URL: 'www.github.com'

- elements:
  - div
  - p
  - button
```

So in the base function for running the tests, which is just called with the options from the cli, we basically load the required files.

```python
def run_tests(
    project_path_str: str,
    calls_path_str: str,
    data_path_str: str,
    continue_on_failure: bool,
    output: str,
) -> None:
    """
    Run the tests.

    Parameters
    ----------
    project_path_str : str
        Path to the project.
    calls_path_str : str
        Path to the calls config.
    data_path_str : str
        Path to the data config.
    continue_on_failure : bool
        Continue tests on error.
    output : str
        Path to the output folder.
    """
    project_path: Path = Path(project_path_str)
    test_tool_logger.info(
        "Running tests for project %s", project_path.as_posix()
    )

    calls_path: Path = project_path.joinpath(calls_path_str)
    test_tool_logger.info("Calls: %s", calls_path.relative_to(project_path))

    data_path: Path = project_path.joinpath(data_path_str)
    if data_path.exists():
        test_tool_logger.info("Data: %s", data_path.relative_to(project_path))

        # Load the data
        data: Dict[str, Any] = load_config_yaml(data_path)
        if data is None:
            data = {}
        for key, value in data.items():
            test_tool_logger.debug("Loaded %s for %s", value, key)
    else:
        test_tool_logger.info("No data file found, using empty data dict.")
        data = {}

    # Set the project path in the data
    data["PROJECT_PATH"] = project_path.as_posix()

    # Check if output is set
    if output:
        # Create a string from datetime in format YYYYMMDD_HHMMSS
        now_str = datetime.now().strftime(output)
        output_path: Path = project_path.joinpath(now_str)
        test_tool_logger.debug(
            "Create output folder %s", output_path.relative_to(project_path)
        )
        output_path.mkdir(parents=True, exist_ok=True)

        # Set the output path in the data
        data["OUTPUT_PATH"] = output_path.absolute().as_posix()

        # Add Handler to logger to write to file
        fh = FileHandler(output_path.joinpath("run.log"))
        fh.setLevel(INFO)
        fh.setFormatter(
            Formatter("%(asctime)s | %(levelname)s | %(name)s | %(message)s")
        )
        test_tool_logger.addHandler(fh)

    # Load the calls
    calls: List[Call] = load_config_yaml(calls_path, True)
    errors = make_all_calls(calls, data, project_path, continue_on_failure)

    if errors == 0:
        test_tool_logger.info("Everything OK")
    else:
        test_tool_logger.error(
            "There occurred %s test_tool_logger.errors while testing,"
            + "please check the logs",
            errors,
        )
        sys.exit(1)
```

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

## Plugins

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

As mentioned above, the plugins handle the actual test steps. Some plugins might be quite simple, some might be more
complex. But the structure of the plugins is always the same.

Some plugins ship with the software, some might be written by the user, or some might be downloaded from the internet.
We require the naming scheme for the plugins to be the same, so that the software can find them. The naming scheme is
***test_tool_[a-zA-Z\\_]\_plugin***.

So if a test step is defined in the **_Configuration-File_** with the **_type_** ***FOO***, the software will look for a
module named ***test_tool_foo_plugin*** to import.

The dependencies for the plugins shipped with the software are installed with the software.

What we do in the code is to import the plugin and save the required functions in a dictionary to cache them for later
usage. Later when I write about the lifecycle of a test, you will see which functions are required per plugin.

The code for loading the plugins looks like:

```python
"""
In this file, we import the necessary modules for the test tool to run.
"""
from importlib import import_module
from logging import getLogger
from types import FunctionType
from typing import Any, Callable, Dict, TypedDict
from copy import deepcopy

# Get the logger
test_tool_logger = getLogger("test-tool")


class CallType(TypedDict):
    """
    Call Type.
    """

    default_call: Dict[str, Any]
    augment_call: Callable
    make_call: Callable


# Plugin Name Templates
PLUGIN_NAME_TEMPLATE: str = "test_tool_${plugin}_plugin"

# Plugin Template
PLUGIN_TEMPLATE: Dict[str, str] = {
    "default_call": "default_${plugin}_call",
    "augment_call": "augment_${plugin}_call",
    "make_call": "make_${plugin}_call",
}

PLUGIN_COMPONENT_TYPES: Dict[str, type] = {
    "default_call": dict,
    "augment_call": FunctionType,
    "make_call": FunctionType,
}

PLUGIN_DEFAULT: CallType = {
    "default_call": {},
    "augment_call": lambda *args, **kwargs: None,
    "make_call": lambda *args, **kwargs: None,
}


def import_plugin(plugin: str, loaded_call_types: Dict[str, CallType]) -> bool:
    """
    Dynamically import the specified plugin as a module.

    Parameters
    ----------
    plugin : str
        Name of the plugin.

    Returns
    -------
    bool
        True if the plugin was loaded successfully, False otherwise.
    """
    try:
        plugin_module = import_module(
            PLUGIN_NAME_TEMPLATE.replace("${plugin}", plugin.lower())
        )
    except ModuleNotFoundError:
        test_tool_logger.error(
            "Plugin %s not found, try to install it with pip install"
            + " test_tool_%s_plugin",
            plugin,
            plugin.lower(),
        )
        return False

    # Create a dict for the loaded plugin
    loaded_plugin: CallType = deepcopy(PLUGIN_DEFAULT)

    # Load the components from the plugin
    to_load: Dict[str, str] = {
        key: val.replace("${plugin}", plugin.lower())
        for key, val in PLUGIN_TEMPLATE.items()
    }
    for key, component in to_load.items():
        try:
            loaded_plugin[key] = getattr(  # type: ignore
                plugin_module, component
            )
        except AttributeError:
            # We can ignore this error, because default value is set
            pass
        if not isinstance(
            loaded_plugin[key], PLUGIN_COMPONENT_TYPES[key]  # type: ignore
        ):
            msg: str = (
                f"Module test_tool_{plugin.lower()}_plugin is not a valid "
                + f"plugin, {component} is not a {PLUGIN_COMPONENT_TYPES[key]}"
            )
            test_tool_logger.error(msg)
            raise AttributeError(msg)

    loaded_call_types[plugin] = loaded_plugin
    return True
```

So as mentioned above, what we do here is to import the plugin and save the required functions in a dictionary to cache
them for later usage.

In Line 62 we use the **_importlib_**[^importlib] to import the plugin. If the plugin is not found, we log an error and
return
False.

If the module is found, we create a dictionary with the required functions and a default value. The default value is
required, since we want to keep the structure for every plugin the same, even if the plugin does not provide the
function.

For now there are only three parts loaded from the plugin. The **_default_call_** is a dictionary which contains the
default values for the call. The **_augment_call_** is a function which can be used to add additional data to the call,
or manipulate it beforehand. The **_make_call_** is the function which actually runs the test step.

This design is quite flexible, since we can add more functions to the plugin if required. In that case we only need to
add a new key to the **_PLUGIN_TEMPLATE_**, the **_PLUGIN_COMPONENT_TYPES_** and the **_PLUGIN_DEFAULT_**. We don't need
to change the code for the plugins.

The ***dictionary*** **_loaded_call_types_** is used to cache the loaded plugins for later usage, it is passed to this
function, so that we actually can store it in the main function for running all test.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

## Lifecycle of a test

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

Now we have all the tests loaded and want to run one after another. But we want to keep this in a structured way, so we
can handle every plugin in the same manner and don't lose too much power for the plugins.

In the previous section we have seen that we have two functions for every plugin. The **_augment_call_** and the
**_make_call_**.

The lifecycle of a test is as follows:

- Check if a test type is given (Otherwise we use ASSERT as default)
- Check if the plugin is loaded (And load it if necessary)
- Merge the default call with the call from the configuration
- Replace the placeholders in the call with the data from the data file[^datafile]
- Run the **_augment_call_** function
- Replace the placeholders in the call again, since the **_augment_call_** function might have added new variables we
  want to replace.
- Run the **_make_call_** function

We run all that function under a try-except block, so that we can catch errors and log them. If the *
*_continue_on_failure_** flag is set, we don't stop the tests, but log the error and continue with the next test.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

## Conclusion

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

The design of the software is quite flexible and easy to extend. The software is designed to be easy configurable and to
be
easy testable. If you want to read more about this project, you will soon find a blog entry about how we wrote a
selenium plugin to run tests with selenium.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

## About this project

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

You can see the project on GitHub [universal-test-tool](https://github.com/jackovsky8/universal-test-tool) and use it
for testing your systems.

[^yaml]: [The Official Yaml Web Site](https://yaml.org/)
[^plugins]: In a blog entry about plugins
[^importlib]: [The Official Python Documentation](https://docs.python.org/3/library/importlib.html)
[^datafile]: We can also store data in this object during other test steps.

```
