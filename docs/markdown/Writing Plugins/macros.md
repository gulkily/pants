---
title: "Macros"
slug: "macros"
excerpt: "Reducing boilerplate in BUILD files."
hidden: false
createdAt: "2020-05-08T04:15:04.126Z"
updatedAt: "2022-05-12T15:59:18.084Z"
---
When to use a macro
-------------------

Macros are useful to reduce boilerplate in BUILD files. For example, if you keep using the same value for a field, you can use a macro. 

However, also consider that introducing new symbols to BUILD files adds some indirection to your codebase, such as making it harder to follow along with the Pants docs. As with any tool, macros should be used judiciously.

Often, you can instead use the [`parametrize`](doc:targets) mechanism:

```python BUILD
shell_tests(
    name="tests",
    shell=parametrize("bash", "zsh"),
)
```

If you instead want to add support for a new language, or do something more complex than a macro allows, create a new [target type](doc:target-api-new-targets).

If you are already using a target type, but need to store additional metadata for your plugin, [add a new field to the target type](doc:target-api-extending-targets).

How to add a macro
------------------

Macros are defined in Python files that act like a normal BUILD file. They have access to all the symbols you normally have registered in a BUILD file, such as all of your target types. 

Macros cannot import other modules, just like BUILD files cannot have import statements.

To define a new macro, add a function with `def` and the name of the new symbol. Usually, the last line of the macro will create a new target, like this:

```python pants-plugins/macros.py
def python2_sources(**kwargs):
    kwargs["interpreter_constraints"] = ["==2.7.*"]
    python_sources(**kwargs)

def python3_sources(**kwargs):
    kwargs["interpreter_constraints"] = [">=3.5"]
    python_sources(**kwargs)
```

Then, add this file to the option `[GLOBAL].build_file_prelude_globs`:

```toml pants.toml
[GLOBAL]
build_file_prelude_globs = ["pants-plugins/macros.py"]
```

Now, in BUILD files, you can use the new macros:

```python project/BUILD
python2_sources(
    name="app_py2",
    sources=["app_py2.py"],
)

python3_sources(
    name="app_py3",
    sources=["app_py3.py"],
)
```

A macro can create multiple targets—although often it's better to use [`parametrize`](doc:targets):

```python pants-plugins/macros.py
def python23_tests(name, **kwargs):
    kwargs.pop("interpreter_constraints", None)

    python_tests(
        name=f"{name}_py2",
        interpreter_constraints=["==2.7.*"],
        **kwargs,
    )
 
    python_tests(
        name=f"{name}_py3",
        interpreter_constraints=[">=3.5"],
        **kwargs,
    )
```

A macro can perform validation:

```python pants-plugins/macros.py
def custom_python_sources(**kwargs):
    if "2.7" in kwargs.get("interpreter_constraints", ""):
        raise ValueError("Python 2.7 is banned!")
    python_sources(**kwargs)
```

A macro can take new parameters to generate the target dynamically. For example:

```python pants-plugins/macros.py
def custom_python_sources(has_type_hints: bool = True, **kwargs):
    if has_type_hints:
        kwargs["tags"] = kwargs.get("tags", []) + ["type_checked"]
    python_sources(**kwargs)
```
```python project/BUILD
custom_python_sources(
    has_type_hints=False,
)
```
