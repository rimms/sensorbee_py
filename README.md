py supports Python module and instance using `PyObject`.

# Set up to link python

Go codes in `py` package use cgo to call `PyObject`, cgo code is here:

"cgolinks.go"

```go
/*
#cgo pkg-config: python-2.7
*/
```

other *.go, for example "pydatetime.go"

```go
/*
#include "Python.h"
#include "datetime.h"
...
*/
```


Currently py package library use pkg-config to link "Python.h". User needs to set up pkg-config and "python-2.7.pc".

* [TODO] currently only support darwin and linux, need to support windows
* [TODO] support python3

## Example to set up pkg-config (with pyenv)

If user uses pyenv, "python-2.7.pc" would be installed at `~/.pyenv/versions/<version>/lib/pkgconfig`, and user can use this file.

Default pkg-config path (`PKG_CONFIG_PATH`) would be `/usr/local/lib/pkgconfig/`, then

```sh
ln -s ~/.pyenv/versions/<version>/lib/pkgconfig/python-2.7.pc /usr/local/lib/pkgconfig/
```

### When occur link object error

When python is installed as static object (*.so),  error will be occurred on building Sensorbee. Please try to re-install Python with enabled shared.

```bash
env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install -v 2.7.9
```

# Default UDS/UDF

## pystate

"lib/sample_module.py"

```python
class SampleClass(object):
    def __init__(self, arg1, arg2='arg2', arg3='arg3', **arg4):
        # initializing
        # blow BQL sample will set like:
        # arg1 = 'arg1'
        # arg2 = 'arg2' # default value
        # arg3 = 'arg3a' # overwritten
        # arg4 = {'new_arg1':1, 'new_arg2':'2'} # variable named arguments

    def sample_method(self, v1, v2, v3):
        # do something

    def write_method(self, value):
        # do something
```

The above python class can be created as SharedState, BQL is written like:

```sql
CREATE STATE sample_module TYPE pystate
    WITH module_path = 'lib', -- optional, default ''
         module_name = 'sample_module', -- required
         class_name = 'SampleClass',  -- required
         write_method = 'write_method', -- optional
         -- rest parameters are used for initializing constructor arguments.
         arg1 = 'arg1',
         arg3 = 'arg3a',
         new_arg1 = 1,
         new_arg2 = '2'
;
```

### pystate_func

UDF query is written like:

```sql
EVAL pystate_func('sample_module', 'sample_method', arg1, arg2, arg3)
;
```

User must make correspond with python `sample_method` arguments with UDF arguments.

### python code

Those UDS creation query and UDF are same as following python code.

```python
import sample_module

# same as CREATE STATE
sm = sample_module.SampleClass(arg1='arg1', arg3='arg3a', new_arg1=1, new_arg2='2')

# same as EVAL
sm.sample_method(arg1, arg2, arg3)
```

### insert into sink

When a pystate is set "write\_method" value, then the state is writable, and if not set "write\_method" then SensorBee will return an error.

[TODO] need to discussion default writable specification.

See SensorBee wiki: [Updating a UDS](https://github.pfidev.jp/sensorbee/sensorbee/wiki/How-to-write-%22stateful%22-User-Defined-Functions#updating-a-uds)

## Default Register

py/pystate supports default registration.

### UDS

User can add original type name pystate. In an application, write like:

```go
import (
    "pfi/sensorbee/py"
)

func init() {
    py.MustRegisterPyUDSCreator("my_uds", "lib", "sample_module",
        "SampleClass", "write_method")
}
```

Then user can use `TYPE my_uds` UDS, like:

```sql
CREATE STATE sample_module TYPE org_uds
    WITH v1 = -1,
         v2 = 'string'
;
```

User can also use "pystate\_func" same as "pystate"

```sql
EVAL pystate_func('sample_module', 'sample_method', arg1, arg2, arg3)
;
```

`MustRegisterPyUDSCreator` is just an alias.


### UDF

User can register a python module method directly.

"lib/sample_module.py"

```python
# module method
def sample_module_method(arg1, arg2):
    # do something
```

In an application, write like:

```go
import (
    "pfi/sensorbee/py"
)

func init() {
    py.MustRegisterPyUDF("my_udf", "lib", "sample_module",
        "sample_module_method")
}
```

User can `my_udf`.

```sql
EVAL my_udf(arg1, arg2)
;
```

This sample query is same as following Python code.

```python
import sample_module

sample_module.sample_module_method(arg1, arg2)
```
