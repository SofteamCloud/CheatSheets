# Python best practices

## Usage 
Typically when we reach for Python we're either writing:

- library code
- pipeline code

| Type            | Purpose                                                       |
| --------------- | ------------------------------------------------------------- |
| _library code_  | Is code with the intent to be imported into other python code |
| _pipeline code_ | Is code with the sole purpose of doing ELT jobs.              |

The style guide provides different guidelines for either.

## 1. Version

Unless the technology you're using (ie, Dataflow) doesn't support it; Use python version 3.7 or greater

<a id="s2-python-language-rules"></a>
<a id="python-language-rules"></a>
## 2 Python Language Rules

<a id="s2.1-lint"></a>
<a id="lint"></a>
### 2.1 Lint

Run `pylint` over your code.

<a id="s2.1.1-definition"></a>
#### 2.1.1 Definition

`pylint` is a tool for finding bugs and style problems in Python source
code. It finds problems that are typically caught by a compiler for less dynamic
languages like C and C++. Because of the dynamic nature of Python, some
warnings may be incorrect; however, spurious warnings should be fairly
infrequent.

You may want to disable C0301/C0330 in pylint since line length/indentation should be handled by our formatting tool black.
You can accomplish this by first creating a pylintrc file.
```
pylint --generate-rcfile > ~/.pylintrc
```
Then adding the following to the disabled list in the [MESSAGES CONTROl] section.
```
[MESSAGES CONTROL]
...
disable=...,
        C0301, # disables warnings about line length
        C0330  # disables warnings about wrong hanging indentation
```

As was just mentioned, the defacto formatting tool for this team is `black` [https://github.com/python/black](https://github.com/python/black).

Simply run
```
black .
```
and it will take care of the rest

<a id="s2.2-pytest"></a>
<a id="pytest"></a>
### 2.2 Pytest

Pytest is the tester of choice against python unit test files.

<a id="s2.2.1-definition"></a>
#### 2.2.1 Definition
https://docs.pytest.org/en/latest/

Install pytest by running:
```
pip install pytest
```

and then run against your test file by running either of the following:

```
pytest test_example.py # For a single file
pytest # This just runs pytest against any test python files it can find in the current directory and subdirectories recursively
```

<a id="s2.3-imports"></a>
<a id="imports"></a>
### 2.3 Imports

* Use `import x` for importing packages and modules.
* Use `from x import y` where `x` is the package prefix and `y` is the module
name with no prefix.
* Use `from x import y as z` if two modules named `y` are to be imported or if
`y` is an inconveniently long name.
* Use `import y as z` only when `z` is a standard abbreviation (e.g., `np` for
`numpy`).

If you are only going to use a specific function or class from a module then
specifically import that.

For example the module `sound.effects.echo` may be imported as follows:

```python
from sound.effects.echo import EchoFilter
...
EchoFilter(input, output, delay=0.7, atten=4)
```

Do not use relative names in imports. Even if the module is in the same package,
use the full package name. This helps prevent unintentionally importing a
package twice.

<a id="s2.3-imports-formatting"></a>
<a id="imports-formatting"></a>
#### 2.3.1 Imports formatting

Use isort to format these for you.
Imports should be on separate lines.

E.g.:

```python
Yes: import os
     import sys
```

```python
No:  import os, sys
```

<a id="s2.4-packages"></a>
<a id="packages"></a>
### 2.4 Packages

Import each module using the full pathname location of the module.

Avoids conflicts in module names or incorrect imports due to the module search
path not being what the author expected.  Makes it easier to find modules.

Imports should be as follows:

Yes:
```python
from doctor.who import jodie
```

No: _(assume this file lives in `doctor/who/` where `jodie.py` also exists)_
```python
import jodie
```

If you're stuck with python 2, you can use the `__future__` package to enforce this:

```python
from __future__ import absolute_import
```

<a id="s2.5-exceptions"></a>
<a id="exceptions"></a>
### 2.5 Exceptions

Use try/except sparingly, it tends to lead to code that is really hard to
follow from a control flow perspective.

DO NOT bloat your try block with statements that will never cause the exception
block to execute. This not only makes it harder to read, but it is also pointless.

Every time you reach for a try, make sure you can prove you need it

#### 2.5.1 Don't use try/except to conceal traces

> Bad:

```python
try:
    code_that_can_raise()
except:
    print "Some error message that isn't as helpful"
```

Every time you do this, a puppy is killed. :dog: :skull:

Hiding stack traces doesn't help in anyway; any error message you write will
not be as helpful as the actual stacktrace.

We can leverage the fact that https://cloud.google.com/error-reporting/ will
capture our stack traces and provide information into them, so just let it break

#### 2.5.2 Rules of thumb

Exceptions must follow certain conditions:

-   Raise exceptions like this: `raise MyError('Error message')` or `raise
    MyError()`. Do not use the two-argument form (`raise MyError, 'Error
    message'`).

-   Make use of built-in exception classes when it makes sense. For example,
    raise a `ValueError` if you were passed a negative number but were expecting
    a positive one. Do not use `assert` statements for validating argument
    values of a public API. `assert` is used to ensure internal correctness, not
    to enforce correct usage nor to indicate that some unexpected event
    occurred. If an exception is desired in the latter cases, use a raise
    statement. For example:

    
    ```python
    Yes:
        if not port:
          raise ConnectionError('Could not connect to service on %d or higher.' % (minimum,))
        assert port >= minimum, 'Unexpected port %d when minimum was %d.' % (port, minimum)
        return port
    ```

    ```python
    No:
        assert minimum >= 1024, 'Minimum port must be at least 1024.'
        port = self._find_next_open_port(minimum)
        assert port is not None
        return port
    ```

-   Never use catch-all `except:` statements, or catch `Exception` or
    `StandardError`, unless you are re-raising the exception or in the outermost
    block in your thread (and printing an error message). Python is very
    tolerant in this regard and `except:` will really catch everything including
    misspelled names, sys.exit() calls, Ctrl+C interrupts, unittest failures and
    all kinds of other exceptions that you simply don't want to catch.

-   Don't attempt to make custom exceptions when built in ones are available.

-   Minimize the amount of code in a `try`/`except` block. The larger the body
    of the `try`, the more likely that an exception will be raised by a line of
    code that you didn't expect to raise an exception. In those cases, the
    `try`/`except` block hides a real error.

-   Use the `finally` clause to execute code whether or not an exception is
    raised in the `try` block. This is often useful for cleanup, i.e., closing a
    file.

-   When capturing an exception, use `as` rather than a comma. For example:
    
    ```python
    try:
      raise Error()
    except Error as error:
      pass
    ```

#### 2.5.3 When capturing Exceptions re-raise using `from`

When you're capturing errors to do something special with the Exceptions,
re-raise the original exception so that the stacktrace specifically points out
that our exception is caused by another

> Bad:

```python
try:
    do_something_that_raises()
except:
    raise Exception("Our Exception")
    # Congrats, you just killed a puppy; John Wick is coming for you
```

> Good:

```python
try:
    do_something_that_raises()
except SpecificError as err:
    # Something special better be happening here say cleaning up resources
    # and then
    raise Exception("Our Exception") from err
```

<a id="s2.6-default-argument-values"></a>
<a id="default-argument-values"></a>
### 2.6 Default Argument Values

Okay in most cases.

Okay to use with the following caveat:

Do not use mutable objects as default values in the function or method
definition.

```python
Yes: def foo(a, b=None):
         if b is None:
             b = []
Yes: def foo(a, b: Optional[Sequence] = None):
         if b is None:
             b = []
Yes: def foo(a, b: Sequence = ()):  # Empty tuple OK since tuples are immutable
         ...
```

```python
No:  def foo(a, b=[]):
         ...
No:  def foo(a, b=time.time()):  # The time the module was loaded???
         ...
No:  def foo(a, b=FLAGS.my_thing):  # sys.argv has not yet been parsed...
         ...
```

<a id="s3-python-style-rules"></a>
<a id="python-style-rules"></a>
## 3 Python Style Rules

<a id="s3.1-semicolons"></a>
<a id="semicolons"></a>
### 3.1 Semicolons

Do not terminate your lines with semicolons, and do not use semicolons to put
two statements on the same line.

<a id="s3.2-line-length"></a>
<a id="line-length"></a>
### 3.2 Line length

In accordance with the black formatter defaults, the maximum line length is *88 characters*.

Exceptions:

-   Long import statements.
-   URLs, pathnames, or long flags in comments.
-   Long string module level constants not containing whitespace that would be
    inconvenient to split across lines such as URLs or pathnames.
-   Pylint disable comments. (e.g.: `# pylint: disable=invalid-name`)

<a id="s3.3-parentheses"></a>
<a id="parentheses"></a>
### 3.3 Parentheses

Use parentheses sparingly.

It is fine, though not required, to use parentheses around tuples. Do not use
them in return statements or conditional statements unless using parentheses for
implied line continuation or to indicate a tuple.

<a id="s3.8-comments"></a>
<a id="comments"></a>
### 3.4 Comments and Docstrings

Use the "Google" style as laid out
[https://sphinxcontrib-napoleon.readthedocs.io/en/latest/](https://sphinxcontrib-napoleon.readthedocs.io/en/latest/)

Example:

```python
class MyClass(object):
    """Summary line.

    Longer information
    """
    def method_name(self, arg1, key=1):
        """Summary here.

        Longer explanation.

        Args:
            arg1 (type): What it's used for

        Keyword Args:
            key (type): What it's used for

        Returns:
            type: Explanation if needed

        Raises:
            ValueError: When `arg1` is bad
        """
        pass

```

For a list of Sections in the Docstring see: https://sphinxcontrib-napoleon.readthedocs.io/en/latest/#docstring-sections

The goal is that someone totally new to the entire codebase should be able to read a docstring and understand what the method/class purpose and function is.
Therefore, if the functionality/purpose is so straightforward that a summary line is sufficient then no need to make up a repetitive body paragraph too.

For unit tests, a module docstring plainly explaining what module/file/etc the test file is testing is fine. No need to add further docstrings to each method
or class in a unit test file since each unit test should already be very brief, test a single use case, and the title of the test should be as descriptive as a
docstring summary anyway.


<a id="comments-in-block-and-inline"></a>
<a id="s3.8.5-comments-in-block-and-inline"></a>
#### 3.4.1 Block and Inline Comments

The final place to have comments is in tricky parts of the code. If you're going
to have to explain it at the next , you should comment it now. Complicated
operations get a few lines of comments before the operations commence.
Non-obvious ones get comments at the end of the line.

```python
# We use a weighted dictionary search to find out where i is in
# the array.  We extrapolate position based on the largest num
# in the array and the array size and then do binary search to
# get the exact number.

if i & (i-1) == 0:  # True if i is 0 or a power of 2.
```

<a id="s3.16-naming"></a>
<a id="naming"></a>
### 3.5 Naming

`module_name`,
`package_name`,
`ClassName`,
`method_name`,
`ExceptionName`,
`function_name`,
`GLOBAL_CONSTANT_NAME`,
`global_var_name`,
`instance_var_name`,
`function_parameter_name`,
`local_var_name`.

Function names, variable names, and filenames should be descriptive; eschew
abbreviation. In particular, do not use abbreviations that are ambiguous or
unfamiliar to readers outside your project, and do not abbreviate by deleting
letters within a word.

Always use a `.py` filename extension.

#### 3.5.1 Don't
Never use hyphens `-`

If not creating a class, don't use "camelCase", instead use "snake_case"

#### 3.5.2 There is no private in python, don't force it

Don't use double underscore (_"dunder"_) names to hide things via name
mangling. This just makes code a pita to test and read and it's not really
private anyway ... just don't.

While Prefixing a name with a single "underscore" does hide somethings from
import statements, who are we trying to protect?

It's much easier to just... not call the code. Add a docstring and tell
developers not to call it

If you have code that shouldn't ever, by no circumstances whatsoever, ever ever
be run by somebody that imports your code, then don't use python use something
else; you're fighting it here

#### 3.5.3 Python isn't Java

You're not limited to one class per file. Write as many classes as you need in
a file, remember that a developer is using the file name to navigate the code
base.

Don't use abstract classes or methods:

> Bad:

```python
from abc import ....
```

https://www.youtube.com/watch?v=31g0YE61PLQ

> Good:

```python
class ConcreteThing: pass
```

### 3.6 Numbers

Use underscores to separate big numbers

> Bad:

```python
num = 1000000
```

> Good:

```python
num = 1_000_000
```

<a id="s3.10-strings"></a>
<a id="strings"></a>
### 3.7 Strings

https://docs.python.org/3/whatsnew/3.6.html#whatsnew36-pep498

> Bad:

```python
val = "string"
val2 = "here"

x = "my {} is {}".format(val, val2)
```

> Good:

```python
val = "string"
val2 = "here"

x = f"my {val} is {val2}"
```

Do not overuse f-strings by using them when they're not necessary

> Bad:

```python
variable = "foobar"

value = "f{variable}"
```

> Good:

```python
variable = "foobar"
```


## 4 Contexts

### 4.1 

Favor using [Context Mangers](https://docs.python.org/3.7/library/contextlib.html#examples-and-recipes) for handling file operations ie: the `with` keyword.

> Bad:

```python
f = open('file', 'r')
# do work
f.close()
```

> Good:

```python
with open('file', 'r') as f:
    # do work
```

### 4.2

Don't use the identifer `file` for context scopes. That is a built-in in python2 and we'll just not use in any version of python to keep consistency.

> Bad:

```python
with open('file', 'r') as file:
    # do work
```

> Good:

```python
with open('file', 'r') as f:
    # do work
```

## 5 Parting Words

*BE CONSISTENT*.

If you're editing code, take a few minutes to look at the code around you and
determine its style. If they use spaces around all their arithmetic operators,
you should too. If their comments have little boxes of hash marks around them,
make your comments have little boxes of hash marks around them too.

The point of having style guidelines is to have a common vocabulary of coding so
people can concentrate on what you're saying rather than on how you're saying
it. We present global style rules here so people know the vocabulary, but local
style is also important. If code you add to a file looks drastically different
from the existing code around it, it throws readers out of their rhythm when
they go to read it. Avoid this.