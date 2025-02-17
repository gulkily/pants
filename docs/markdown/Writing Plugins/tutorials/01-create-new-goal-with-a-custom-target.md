# Introduction

In this tutorial, you'll learn the basics needed to get started writing a plugin. You will create a new goal, `project-version`, which will tell you the version (retrieved from the `VERSION` text file) of a particular project in your monorepository. You will learn how to create a new custom target to refer to the `VERSION` file, how to author a new goal, and, most importantly, how to connect rules and targets. You can follow along this tutorial in your own repository; you only need to be on a recent version of Pants and have a `VERSION` file containing a version string e.g. `1.2.3`. If you do not have a repository with Pants enabled yet, you can use [this example Python repository](https://github.com/pantsbuild/example-python/) to work on the plugin.

## Registering a plugin

We'll be writing an [in-repo plugin](https://www.pantsbuild.org/docs/plugins-overview#in-repo-plugins), and expect 
you to have the `pants-plugins/project_version` directory as well as the `pants.toml` file with this configuration:

```toml pants.toml
# Specifying the path to our plugin's top-level folder using the `pythonpath` option:
pythonpath = ["%(buildroot)s/pants-plugins"]

backend_packages = [
  "pants.backend.python",
   ...
  "project_version",
]
```

## Creating a new target

Once you have become familiar with the [core concepts of Targets and Fields](https://www.pantsbuild.org/docs/target-api-concepts), you are ready to [create an own custom target](https://www.pantsbuild.org/docs/target-api-new-targets) that will represent the `VERSION` file:

```python pants-plugins/project_version/targets.py
from pants.engine.target import COMMON_TARGET_FIELDS, SingleSourceField, Target


class ProjectVersionTarget(Target):
    alias = "version_file"
    core_fields = (*COMMON_TARGET_FIELDS, SingleSourceField)
    help = "A project version target representing the VERSION file."
```

Our target has some common target fields such as `tags` and `description` available via the `COMMON_TARGET_FIELDS`; including those fields in your targets may be convenient if you decide to use tags and provide a description later. In addition, it also has the [`source` field](https://www.pantsbuild.org/docs/rules-api-and-target-api#sourcesfield) which will be used to provide path to the project's `VERSION` file. 

We could [add a custom field](https://www.pantsbuild.org/docs/target-api-new-fields) to provide a file path, however, there are multiple advantages to  using the `source` field. You will learn more about them in the following tutorials. 

In order to start using a target, you only need to register it: 

```python pants-plugins/project_version/register.py
from typing import Iterable

from pants.engine.target import Target
from project_version.target_types import ProjectVersionTarget


def target_types() -> Iterable[type[Target]]:
    return [ProjectVersionTarget]
```

You can now run `./pants help version_file` to learn more about the target:

```
❯ ./pants help version_file  

`version_file` target
---------------------

A project version target representing the VERSION file.


Activated by project_version
Valid fields:

source
    type: str
    required

    A single file that belongs to this target.
    
    Path is relative to the BUILD file's directory, e.g. `source='example.ext'`.

...
```

You can now also add a target to a `myapp/BUILD` file:

```python
version_file(
    name="main-project-version", 
    source="VERSION",
)
```

Since you have registered the target, Pants will be able to "understand" it:

```text
$ ./pants peek myapp:main-project-version
[
  {
    "address": "myapp:main-project-version",
    "target_type": "version_file",
    "dependencies": [],
    "description": null,
    "source_raw": "VERSION",
    "sources": [
      "myapp/VERSION"
    ],
    "tags": null
  }
]
```

## Creating a goal

[Goals](https://www.pantsbuild.org/docs/goals) are the commands that Pants runs such as `fmt` or `lint`. Writing a plugin doesn't necessarily mean adding a new goal. Most users would likely only want to enrich their build metadata with new kinds of targets or extend behavior of existing Pants goals. See [Common plugin tasks](https://www.pantsbuild.org/docs/common-plugin-tasks) to learn more.

For the purposes of our tutorial, to be able to get a project version number (using a target we've just created), we need to [create a new goal](https://www.pantsbuild.org/docs/rules-api-goal-rules). The code below is the boilerplate necessary to create a goal, so it's not really necessary to understand how, for instance, subsystems work right now. The function decorated with the `@goal_rule` can be named anything, but it's helpful for the name to represent the functionality your goal provides. To make your goal part of the plugin's interface, add it to the `rules` function in the `register.py` module.

```python pants-plugins/project_version/rules.py
from pants.engine.goal import Goal, GoalSubsystem
from pants.engine.rules import collect_rules, goal_rule


class ProjectVersionSubsystem(GoalSubsystem):
    name = "project-version"
    help = "Show representation of the project version from the `VERSION` file."


class ProjectVersionGoal(Goal):
    subsystem_cls = ProjectVersionSubsystem


@goal_rule
async def goal_show_project_version() -> ProjectVersionGoal:
    return ProjectVersionGoal(exit_code=0)


def rules():
    return collect_rules()
```
```python pants-plugins/project_version/register.py
from typing import Iterable

import project_version.rules as project_version_rules
from pants.engine.target import Target
from project_version.target_types import ProjectVersionTarget


def target_types() -> Iterable[type[Target]]:
    return [ProjectVersionTarget]


def rules():
    return [*project_version_rules.rules()]
```

You can now run `./pants project-version` to confirm the command exits with the exit code `0`.

At this point, we are ready to do something useful with the new target of ours. Goals generally run on targets, so they need to be passed as an argument in the command line. For instance, to format the `myproject` directory targets, you would run `./pants fmt myproject`. To get the version of a project in your repository, it makes sense to pass to the `project-version` goal a project directory containing the `version_file` definition.

To make a target passed as an argument accessible in the goal rule, we pass the [`Targets`](https://www.pantsbuild.org/docs/rules-api-goal-rules#how-to-operate-on-targets) as input arguments of the function along with the [`Console`](https://www.pantsbuild.org/docs/rules-api-goal-rules#console-output-to-stdoutstderr) object so that we can print the details of our target in the user terminal:

```python
@goal_rule
async def goal_show_project_version(console: Console, targets: Targets) -> ProjectVersionGoal:
    # since we don't know what targets will be passed (e.g. `::`), we want to keep only `version_file` targets
    targets = [tgt for tgt in targets if tgt.alias == ProjectVersionTarget.alias]
    for target in targets:
        console.print_stdout(target.address.metadata())
    return ProjectVersionGoal(exit_code=0)

```

Having the following directory structure:

```text
myapp
├── BUILD
└── VERSION
```

we are ready to inspect our new target:

```text
$ ./pants project-version myapp
{'address': 'myapp:main-project-version'}
```

## Writing a rule

You can think of the `@goal_rule` as of the `main` function in your Python program where you would call various functions that your program needs to complete. For auxiliary code, it makes sense to place it into standalone functions which is what `@rule`s are for.

Let's create a rule that will return a data structure that we'll use to represent our project version. [Data classes](https://www.pantsbuild.org/docs/rules-api-concepts#dataclasses) work really well with Pants engine, so let's create one: 

```python
@dataclass(frozen=True)
class ProjectVersionFileView:
    path: str
    version: str
```

This is what our `@rule` function would return (for now without actually reading the `VERSION` file):

```python
@rule
async def get_project_version_file_view(
    target: ProjectVersionTarget,
) -> ProjectVersionFileView:
    return ProjectVersionFileView(
        path="path", version="1.2.3"
    )
```

Now, we have our `@goal_rule`, but we cannot call the `get_project_version_file_view` function; it's Pants that will determine that a rule is used and will make a function call. Well, what should you do to tell Pants you need that rule executed? You should make a function call that:
* passes an object of the type that matches the type of the rule's input arguments
* requests an object of the type that a rule returns (you can see that in a type hint)

For this, you can use [`Get`](https://www.pantsbuild.org/docs/rules-api-concepts#await-get---awaiting-results-in-a-rule-body):

```python
@goal_rule
async def goal_show_project_version(console: Console, targets: Targets) -> ProjectVersionGoal:
    targets = [tgt for tgt in targets if tgt.alias == ProjectVersionTarget.alias]
    for target in targets:
        project_version = await Get(ProjectVersionFileView, ProjectVersionTarget, target)
        console.print_stdout(project_version)
    return ProjectVersionGoal(exit_code=0)
```

Understanding that calling `Get()` is what causes a particular `@rule` to be executed is essential! It may feel awkward that you cannot run your function. However, by using the `Get()`, you are asking Pants to run your rule, and only knowing this will get you quite far! 

Compare this `Get()` call with the rule signature:

```python
# requesting an object of type "ProjectVersionFileView", passing an object of type "ProjectVersionTarget" in the variable "target"
Get(ProjectVersionFileView, ProjectVersionTarget, target)
```

```python
# it requires an object of type "ProjectVersionTarget" and will return an object of type "ProjectVersionFileView"
@rule
async def get_project_version_file_view(target: ProjectVersionTarget) -> ProjectVersionFileView: ...
```

> 📘 Understanding the requests and rules signatures
> In our basic usage, there's a 1:1 match between the `Get(output: B, input: A, obj)` request and the `@rule(input: A) -> B` function signature. This doesn't have to be the case! When you make a request (providing an input type and asking for an output type), Pants looks at all the [rules in the graph](https://www.pantsbuild.org/docs/rules-api-concepts#the-rule-graph) to find a way from the input to the output using all the available rules. 

> Let's consider a following scenario where you have a few `@rule`s and a `Get()` request:

> ```python
> @rule
> async def rule1(A) -> B: ...
> 
> @rule
> async def rule2(B) -> C: ...
> 
> @goal_rule
> async def main(...):
>     result = await Get(C, A, obj)
> ```

> With the following suite of rules, Pants will "figure out" that in order to return `C`, it's necessary to call `rule1` first to get `B` and then once there's `B`, call `rule2` to get C. This means you can focus on writing individual rules and leave the hard work of finding out the right order of calls that will need happen to Pants!   

The `project-version` Pants goal now shows some useful information -- the target path along with a dummy version. This means our `@rule` was run!

```
$ ./pants project-version myapp
ProjectVersionFileView(path='myapp:main-project-version', version='1.2.3')
```

You would normally expect for a project to have only a single `version_file` target declared, so as an improvement, we could raise an exception if there are multiple targets of this type found within a single project. This is something we'll do in the following tutorials.

## Reading the `VERSION` file

Let's read the `VERSION` file and print the version number in the terminal. The `source` field of our target needs to be ["hydrated"](https://www.pantsbuild.org/docs/rules-api-and-target-api#sourcesfield). [Reading a file](https://www.pantsbuild.org/docs/rules-api-file-system) is pretty straightforward as well. We use `Get()` to transform our inputs as needed. Knowing what class you need to request may be tricky, so make sure to review the documentation, and ask for help if you are stuck!    

```python
@rule
async def get_project_version_file_view(
    target: ProjectVersionTarget,
) -> ProjectVersionFileView:
    sources = await Get(HydratedSources, HydrateSourcesRequest(target[SourcesField]))
    digest_contents = await Get(DigestContents, Digest, sources.snapshot.digest)
    file_content = digest_contents[0]    
    return ProjectVersionFileView(
        path=file_content.path, version=file_content.content.decode("utf-8").strip()
    )
```

If the `@goal_rule` would receive multiple `version_file` targets (which may happen if user would run the goal for multiple projects or provide a recursive glob pattern such as `::`), it would be required to iterate over the list of targets. For efficiency, it is generally encouraged to replace the `Get()` calls in the `for` loop with a [`MultiGet()` call](https://www.pantsbuild.org/docs/rules-api-tips#tip-use-multiget-for-increased-concurrency):

```python
@goal_rule
async def goal_show_project_version(console: Console, targets: Targets) -> ProjectVersionGoal:
    targets = [tgt for tgt in targets if tgt.alias == ProjectVersionTarget.alias]
    results = await MultiGet(
        Get(ProjectVersionFileView, ProjectVersionTarget, target) for target in targets
    )
    for result in results:
        console.print_stdout(str(result))
    return ProjectVersionGoal(exit_code=0)
```

## Putting it all together

Let's get all of this code in one place and see what happens!

```python pants-plugins/project_version/rules.py
from dataclasses import dataclass

from pants.engine.console import Console
from pants.engine.fs import DigestContents
from pants.engine.goal import Goal, GoalSubsystem
from pants.engine.internals.native_engine import Digest
from pants.engine.internals.selectors import Get, MultiGet
from pants.engine.rules import collect_rules, goal_rule, rule
from pants.engine.target import (HydratedSources, HydrateSourcesRequest,
                                 SourcesField, Targets)
from project_version.target_types import ProjectVersionTarget


@dataclass(frozen=True)
class ProjectVersionFileView:
    path: str
    version: str


@rule
async def get_project_version_file_view(
    target: ProjectVersionTarget,
) -> ProjectVersionFileView:
    sources = await Get(HydratedSources, HydrateSourcesRequest(target[SourcesField]))
    digest_contents = await Get(DigestContents, Digest, sources.snapshot.digest)
    file_content = digest_contents[0]
    return ProjectVersionFileView(
        path=file_content.path, version=file_content.content.decode("utf-8").strip()
    )


class ProjectVersionSubsystem(GoalSubsystem):
    name = "project-version"
    help = "Show representation of the project version from the `VERSION` file."


class ProjectVersionGoal(Goal):
    subsystem_cls = ProjectVersionSubsystem


@goal_rule
async def goal_show_project_version(
    console: Console, targets: Targets
) -> ProjectVersionGoal:
    targets = [tgt for tgt in targets if tgt.alias == ProjectVersionTarget.alias]
    results = await MultiGet(
        Get(ProjectVersionFileView, ProjectVersionTarget, target) for target in targets
    )
    for result in results:
        console.print_stdout(str(result))
    return ProjectVersionGoal(exit_code=0)


def rules():
    return collect_rules()
```
```python pants-plugins/project_version/targets.py
from pants.engine.target import COMMON_TARGET_FIELDS, SingleSourceField, Target


class ProjectVersionTarget(Target):
    alias = "version_file"
    core_fields = (*COMMON_TARGET_FIELDS, SingleSourceField)
    help = "A project version target representing the VERSION file."
```
```python pants-plugins/project_version/register.py
from typing import Iterable

import project_version.rules as project_version_rules
from pants.engine.target import Target
from project_version.target_types import ProjectVersionTarget


def target_types() -> Iterable[type[Target]]:
    return [ProjectVersionTarget]


def rules():
    return [*project_version_rules.rules()]
```

Running our goal:

```
$ /pants project-version myapp
ProjectVersionFileView(path='myapp/VERSION', version='0.0.1')
```

The `VERSION` file was read and its contents is shown in the console. Congratulations, you have now finished writing your first plugin! 

There are a few things that we could do to improve it, though. We may want to check that the version string follows a semver convention, let user see the version in the console as a JSON object if desired, or show the version number string when exploring the `version_file` target via the `peek` Pants goal. This is something we'll do in the following tutorials! 
