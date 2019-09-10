# GDB dashboard

GDB dashboard is a standalone `.gdbinit` file written using the [Python API][] that enables a modular interface showing relevant information about the program being debugged. Its main goal is to reduce the number of GDB commands needed to inspect the status of current program thus allowing the developer to primarily focus on the control flow.

![Screenshot](https://raw.githubusercontent.com/wiki/cyrus-and/gdb-dashboard/Screenshot.png)

[Python API]: https://sourceware.org/gdb/onlinedocs/gdb/Python-API.html

## Quickstart

Just place [`.gdbinit`][] in your home directory, for example with:

```
wget -P ~ https://git.io/.gdbinit
```

Optionally install [Pygments][] to enable syntax highlighting:

```
pip install pygments
```

Then debug as usual, the dashboard will appear automatically when the inferior program stops.

Keep in mind that no GDB command has been redefined, instead all the features are available via the main `dashboard` command (see `help dashboard`).

The [wiki][] also can be useful as it contains some common use cases.

[`.gdbinit`]: https://raw.githubusercontent.com/cyrus-and/gdb-dashboard/master/.gdbinit
[Pygments]: http://pygments.org/
[wiki]: https://github.com/cyrus-and/gdb-dashboard/wiki

## Requirements

GDB dashboard requires at least GDB 7.7 compiled with Python 2.7 in order to work properly, see [#1][] for more details and possible workarounds. To find the Python version used by GDB run:

```
gdb --batch -ex 'python import sys; print(sys.version)'
```

Make sure that the system locale is configured to use UTF-8, in most cases it already is, otherwise (in case of `UnicodeEncodeError` errors) a simple solution is to export the following environment variable:

```
export LC_CTYPE=C.UTF-8
```

On Windows the [`windows-curses`][] Python package is needed in order to obtain the correct terminal size.

GDB dashboard is not meant to work seamlessly with additional front ends, e.g., TUI, Nemiver, QtCreator, etc. either instruct the front end to not load the `.gdbinit` file or load the dashboard manually.

[#1]: https://github.com/cyrus-and/gdb-dashboard/issues/1
[`windows-curses`]: https://pypi.org/project/windows-curses/

## Configuration

Files in `~/.gdbinit.d/` are executed in alphabetical order, but the preference is given to Python files. If there are subdirectories, they are walked recursively. The idea is to keep separated the custom modules definition from the configuration itself.

By convention, the *main* configuration file should be placed in `~/.gdbinit.d/` (say `~/.gdbinit.d/init`) and can be used to tune the dashboard styles and modules configuration but also the usual GDB parameters.

## Custom modules

A custom module is a Python class that inherit the `Dashboard.Module` class and define some methods specified by the below API. The name of a module is automatically obtained by the class name.

Modules are instantiated once at initialization time and kept during the whole the GDB session.

Optionally, a module may include a description which will appear in the GDB help system by specifying a Python docstring for the class. By convention the first line should contain a brief description.

### API

#### label

```python
def label(self):
```

Return the module label which will appear in the divider.

#### lines

```python
def lines(self, term_width, term_height, style_changed):
```

The available terminal dimensions are passed and if `style_changed` is `True` then it means that some attributes have changed since the last time so the implementation may want to update its status.

Return a list of strings which will form the module content. When a module is temporarily unable to produce its content, it should return an empty list; its divider will then use the styles with the `off` qualifier.

#### attributes

```python
def attributes(self):
```

Return a dictionary in which the key is the attribute name and the value is another dictionary with items:

- `default` is the initial value for this attribute;

- `doc` is the optional documentation of this attribute which will appear in the GDB help system;

- `name` is the name of the attribute of the Python object (defaults to the key value);

- `type` is the Python type of this attribute defaulting to the `str` type, it is used to coerce the value passed as an argument to the proper type, or raise an exception;

- `check` is an optional control callback which accept the coerced value and returns `True` if the value satisfies the constraint and `False` otherwise.

Those attributes can be accessed from the implementation using instance variables named `name`.

#### commands

```python
def commands(self):
```

Return a dictionary in which the key is the attribute name and the value is another dictionary with items:

- `action` is the callback to be executed which accepts the raw input string from the GDB prompt, exceptions in these functions will be shown automatically to the user;

- `doc` is the documentation of this command which will appear in the GDB help system;

- `completion` is the optional completion policy, one of the `gdb.COMPLETE_*` constants defined in the [reference manual][completion].

[completion]: https://sourceware.org/gdb/onlinedocs/gdb/Commands-In-Python.html

### Common functions

A number of auxiliary common functions are defined in the global scope, they can be found in the provided `.gdbinit` and concern topics like [ANSI][] output, divider formatting, conversion callbacks, etc. See the bundled default modules for some usage examples.

### Common styles

There are general purpose [ANSI][] styles defined for convenience and used by modules:

- `style_selected_1`;
- `style_selected_2`;
- `style_low`;
- `style_high`;
- `style_error`;
- `style_critical`.

Use them in conjunction with the `ansi` common function, for example:

```python
my_error = ansi('my_error', R.style_error)
```

### Example

Default modules should provide a good example, but here is a simple module which may be used as a template for new custom modules. This module allows the programmer to note down some snippets of text during the debugging session.

```python
from datetime import datetime

class Notes(Dashboard.Module):
    '''Simple user-defined notes.'''

    def __init__(self):
        self.notes = []

    def label(self):
        return 'Notes'

    def lines(self, term_width, term_height, style_changed):
        out = []
        for index, (ts, note) in enumerate(self.notes):
            # format the index
            index = ansi(str(index), R.style_selected_1)
            line = '[{}]'.format(index)
            # optionally add the timestamp
            if self.timestamp:
                ts = ts.strftime('%Y-%m-%d %H:%M:%S')
                line += ' {} '.format(ansi(ts, R.style_high))
            # finally add the note
            line += ' {}'.format(note)
            out.append(line)
        return out

    def add(self, arg):
        if arg:
            self.notes.append((datetime.now(), arg))
        else:
            raise Exception('Cannot add an empty note')

    def drop(self, arg):
        if arg:
            index = int(arg)
            del self.notes[index]
        else:
            raise Exception('Specify the note index')

    def clear(self, arg):
        self.notes = []

    def commands(self):
        return {
            'add': {
                'action': self.add,
                'doc': 'Add a note.'
            },
            'drop': {
                'action': self.drop,
                'doc': 'Remove a note by index.'
            },
            'clear': {
                'action': self.clear,
                'doc': 'Remove all the notes.'
            }
        }

    def attributes(self):
        return {
            'timestamp': {
                'doc': 'Show the timestamp along with the note.',
                'default': True,
                'type': bool
            }
        }
```

To use the above just save it in a Python file, say `notes.py`, inside `~/.gdbinit.d/`. The following commands will be available (together with the corresponding help):

```
dashboard notes
dashboard notes add
dashboard notes drop
dashboard notes clear
dashboard notes -style timestamp
```

[ANSI]: https://en.wikipedia.org/wiki/ANSI_escape_code
