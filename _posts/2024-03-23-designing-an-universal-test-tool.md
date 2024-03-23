---
layout: post
title:  Designing An Universal Test Tool!
layout: post
date:   2024-03-23 14:37:17 +0100
tags:   python universal-test-tool
---

The problem I try to solve here, is to run integration and system tests for a lot of different type of services which either previously have been made manually, with different tools or with copy and paste scripts.

The idea first started with another scripts which handle different task by a configuration file, so the script could stay the same for all tests and only the configuration had to be rewritten.

This approach really soon came to an dead end, since the script was not readable due to it's complex structure and also not really testable. So every little change on one end broke another end.

So the whole script had to be designed from scratch to be more robust and still keep the flexibility which is required.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

# The design

{: .mt-4 .mb-0 }

<!-- markdownlint-restore -->

The design approach was to have a base script which builds a kind of a framework and starts the actual tests in plugins. With this approach the base script does not need to know about the tests.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

## Requirements

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

1. A CLI to start the program
2. The tests should be easy configurable
3. TODO

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

### Designing The CLI

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

It is quite easy to create a cli with the element from the python standard library [argparse](https://docs.python.org/3/library/argparse.html).
So we only declare what we need, add a text for printing the help and if required some default values.

So the easiest way to start the cli should be, that we access the folder where the testconfiguration is stored and run **_test-tool_**.
If the naming of the configuration files is right, the no further arguments are needed.

As too see in the code below, there are to different config files per test. The **_Configuration-File_** `config.yaml` and the **_Data-File_** `data.yaml`. The **_Configuration-File_** contains the test steps which we want to run. The **_Data-File_** contains complementary data. The ides behind this separation is, that if you want to run the tests in multiple environments you just add another **_Data-File_** and there is no need to change the test steps itself.

The argument **_project_** makes it possible to start a test from everywhere. The path given here will be used as **_cwd_**. So that we can use relative urls in the test configuration.

The other arguments are pretty self explanatory, so we won't discuss them here furter.

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

### Confifguring The Test

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

Since the requirement for this tool was that the tests are eays configurable, the decision for yaml[^yaml] was made. Yaml is easy read and writeable for humans and is powerful enough to design complex test cases.

The basic idea was we want to run one run the test steps one after another. Sometimes it migth be faster to run tests in parallel, but since this program is designed for complex test cases which not necassarily are idempotent we want to keep the order.

So basically the **_Configuration-File_** contains a list of test steps. A test step contains the **_type_** and the **_call_**. The **_type_** defines the plugin with which the test step will be executed. And the **_call_** containes detailed information for the execution. Even if you will see[^plugins], that we take an effort to keep the structure for every call similar, it might look quite different for every plugin.

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
URL: www.github.com

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
            "There occured %s test_tool_logger.errors while testing,"
            + "please check the logs",
            errors,
        )
        sys.exit(1)
```

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

## About this project

You can see the project on GitHub [universal-test-tool](https://github.com/jackovsky8/universal-test-tool) and use it for testing your systems.

{: data-toc-skip='' .mt-4 .mb-0 }

<!-- markdownlint-restore -->

[^yaml]: [The Official Yaml Web Site](https://yaml.org/)
[^plugins]: In a blog entry about plugins
