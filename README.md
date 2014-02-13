# Wrap package #

The __wrap__ package helps you to automate the generation of Lua/C wrappers
around existing C functions, such that these functions would be callable
from Lua. This package is used by the __torch__ package, but does not depend on
anything, and could be used by anyone using Lua.

__DISCLAIMER__ Before going any further, we assume the reader has a good
knowledge of how to interface C functions with Lua. A good start would be
the [Lua reference manual](http://www.lua.org/manual/5.1), or the book
[Programming in Lua](http://www.inf.puc-rio.br/~roberto/pil2).

As an example is often better than lengthy explanations, let's consider the
case of a function
```c
int numel(THDoubleTensor *t);
```
which returns the number of elements of `t`.
Writing a complete wrapper of this function would look like:
```c
static int wrapper_numel(lua_State *L)
{
  THDoubleTensor *t;

  /* always good to check the number of arguments */
  if(lua_gettop(L) != 1)
    error("invalid number of arguments: <tensor> expected");

  /* check if we have a tensor on the stack */
  /* we use the luaT library, which deals with Torch objects */
  /* we assume the torch_DoubleTensor_id has been already initialized */
  t = luaT_checkudata(L, 1, torch_DoubleTensor_id);

  /* push result on stack */
  lua_pushnumber(L, numel(t));

  /* the number of returned variables */
  return 1;
}
```

For anybody familiar with the Lua C API, this should look very simple (and
_it is simple_, Lua has been designed for that!). Nevertheless, the
wrapper contains about 7 lines of C code, for a quite simple
function. Writing wrappers for C functions with multiple arguments, where
some of them might be optional, can become very quickly a tedious task. The
__wrap__ package is here to help the process. Remember however that even
though you might be able to treat most complex cases with __wrap__,
sometimes it is also good to do everything by hand yourself!

## High Level Interface ##

__wrap__ provides only one class: `CInterface`. Considering our easy example, a typical usage
would be:
```lua
require 'wrap'

interface = wrap.CInterface.new()

interface:wrap(
   "numel", -- the Lua name
   "numel", -- the C function name, here the same
   -- now we describe the 'arguments' of the C function
   -- (or possible returned values)
   {
      {name="DoubleTensor"},
      {name="int", creturned=true} -- this one is returned by the C function
   }
)

print(interface:tostring())
```
`CInterface` contains only few methods. [wrap()](#CInterface.wrap) is
the most important one. [tostring()](#CInterface.tostring) returns a
string containing all the code produced until now.  The wrapper generated
by __wrap__ is quite similar to what one would write by hand:
```c
static int wrapper_numel(lua_State *L)
{
  int narg = lua_gettop(L);
  THDoubleTensor *arg1 = NULL;
  int arg2 = 0;
  if(narg == 1
     && (arg1 = luaT_toudata(L, 1, torch_DoubleTensor_id))
    )
  {
  }
  else
    luaL_error(L, "expected arguments: DoubleTensor");
  arg2 = numel(arg1);
  lua_pushnumber(L, (lua_Number)arg2);
  return 1;
}
```

We know describe the methods provided by `CInterface`.

<a name="CInterface.new"/>
### new() ###

Returns a new `CInterface`.

<a name="CInterface.wrap"/>
### wrap(luaname, cfunction, arguments, ...) ###

Tells the `CInterface` to generate a wrapper around the C function
`cfunction`. The function will be called from Lua under the name
`luaname`. The Lua _list_ `arguments` must also be provided. It
describes _all_ the arguments of the C function `cfunction`.
Optionally, if the C function returns a value and one would like to return
it in Lua, this additional value can be also described in the argument
list.
```lua
   {
      {name="DoubleTensor"},
      {name="int", creturned=true} -- this one is returned by the C function
   }
```

Each argument is described also as a list. The list must at least contain
the field `name`, which tells to `CInterface` what type of argument you
want to define. In the above example,
```lua
{name="DoubleTensor"}
```
indicates to `CInterface` that the first argument of `numel()` is of type `DoubleTensor`.

Arguments are defined into a table `CInterface.argtypes`, defined at the
creation of the interface.  Given a `typename`, the corresponding field
in `interface.argtypes[typename]` must exist, such that `CInterface`
knows how to handle the specified argument. A lot of types are already
created by default, but the user can define more if needed, by filling
properly the `argtypes` table. See the section [[#CInterface.argtypes]]
for more details about defined types, and
[how to define additional ones](#CInterface.userargtypes).

#### Argument fields ####

Apart the field `name`, each list describing an argument can contain several optional fields:

`default`: this means the argument will optional in Lua, and the argument will be initialized
with the given default value if not present in the Lua function call. The `default` value might
have different meanings, depending on the argument type (see [[#CInterface.argtypes]] for more details).

`invisible`: the argument will invisible _from Lua_. This special option requires `default` to be set,
such that `CInterface` knows by what initialize this invisible argument.

`returned`: if set to `true`, the argument will be returned by the Lua function. Note that several
values might be returned at the same time in Lua.

`creturned`: if `true`, tells to `CInterface` that this 'argument' is
in fact the value returned by the C function.  This 'argument' cannot have
a `default` value. Also, as in C one can return only one value, only one
'argument' can contain this field! Mixing arguments which are `returned`
and arguments which are `creturned` with `CInterface` is not
recommended: use with care.

While these optional fields are generic to any argument types, some types might define additional optional fields.
Again, see [[#CInterface.argtypes]] for more details.

#### Handling multiple variants of arguments ####

Sometimes, one cannot describe fully the behavior one wants with only a set of possible arguments.
Take the example of the `cos()` function: we might want to apply it to a number, if the given argument
is a number, or to a Tensor, if the given argument is a Tensor.

`wrap()` can be called with extra pairs of `cname, args` if needed. (There are no limitations on the number extra paris).
For example, if you need to handle three cases, it might be
```lua
interface:wrap(luaname, cname1, args1, cname2, args2, cname3, args3)
```
For each given C function name `cname`, the corresponding argument list `args` should match.
As a more concrete example, here is a way to generate a wrapper for `cos()`, which would handle both numbers
and DoubleTensors.
```lua
interface:wrap("cos", -- the Lua function name

"THDoubleTensor_cos", { -- C function called for DoubleTensor
{name="DoubleTensor", default=true, returned=true}, -- returned tensor (if not present, we create an empty tensor)
{name="DoubleTensor"} -- input tensor
},

"cos", { -- the standard C math cos function
{name="double", creturned="true"}, -- returned value
{name="double"} -- input value
}
)
```

<a name="CInterface.print"/>
### print(str) ###

Add some hand-crafted code to the existing generated code. You might want to do that if your wrapper
requires manual tweaks. For e.g., in the example above, the "id" related to `torch.DoubleTensor`
needs to be defined beforehand:
```lua
interface:print([[
const void* torch_DoubleTensor_id;
]])
```

<a name="CInterface.luaname2wrapname"/>
### luaname2wrapname(name) ###

This method defines the name of each generated wrapping function (like
`wrapper_numel` in the example above), given the Lua name of a function
(say `numel`). In general, this has little importance, as the wrapper is
a static function which is not going to be called outside the scope of the
wrap file. However, if you generate some complex wrappers, you might want
to have a control on this to avoid name clashes. The default is
```lua
function CInterface:luaname2wrapname(name)
   return string.format("wrapper_%s", name)
end
```
Changing it to something else can be easily done with (still following the example above)
```lua
function interface:luaname2wrapname(name)
   return string.format("my_own_naming_%s", name)
end
```

### register(name) ###

Produces C code defining a
[luaL_Reg](http://www.lua.org/manual/5.1/manual.html#luaL_Reg) structure
(which will have the given `name`). In the above example, calling
```lua
interface:register('myfuncs')
```
will generate the following additional code:
```c
static const struct luaL_Reg myfuncs [] = {
  {"numel", wrapper_numel},
  {NULL, NULL}
};
```

This structure is meant to be passed as argument to
[luaL_register](http://www.lua.org/manual/5.1/manual.html#luaL_register),
such that Lua will be aware of your new functions. For e.g., the following
would declare `mylib.numel` in Lua:
```lua
interface:print([[
luaL_register(L, "mylib", myfuncs);
]])
```

<a name="CInterface.tostring"/>
### tostring() ###

Returns a string containing all the code generated by the `CInterface`
until now. Note that the history is not erased.

<a name="CInterface.tofile"/>
### tofile(filename) ###

Write in the file (named after `filename`) all the code generated by the
`CInterface` until now. Note that the history is not erased.

<a name="CInterface.clearhhistory"/>
### clearhistory() ###

Forget about all the code generated by the `CInterface` until now.

<a name="CInterface.argtypes"/>
## Argument Types ##

Any `CInterface` is initialized with a default `argtypes` list, at
creation. This list tells to `CInterface` how to handle type names given
to the [wrap()](#CInterface.wrap) method. The user can add more types to
this list, if wanted (see [the next section](#CInterface.userargtypes)).

### Standard C types ###
Standard type names include `unsigned char`, `char`, `short`,
`int`, `long`, `float` and `double`. They define the corresponding
C types, which are converted to/from
[lua_Number](http://www.lua.org/manual/5.1/manual.html#lua_Number).

Additionaly, `byte` is an equivalent naming for `unsigned char`, and
`boolean` is interpreted as a boolean in Lua, and an int in C.

`real` will also be converted to/from a `lua_Number`, while assuming that
it is defined in C as `float` or `double`.

Finally, `index` defines a long C value, which is going to be
automatically incremented by 1 when going from C to Lua, and decremented by
1, when going from Lua to C. This matches Lua policy of having table
indices starting at 1, and C array indices starting at 0.

For all these number values, the `default` field (when defining the
argument in [wrap()](#CInterface.wrap)) can take two types: either a
number or a function (taking the argument table as argument, and returning a string).

Note that in case of an `index` type, the given default value (or result
given by the default initialization function) will be decremented by 1 when
initializing the corresponging C `long` variable.

Here is an example of defining arguments with a default value:
```lua
{name="int", default=0}
```
defines an optional argument which will of type `int` in C (lua_Number in Lua), and will take
the value `0` if it is not present when calling the Lua function. A more complicated (but typical) example
would be:
```lua
{name="int", default=function(arg)
                       return string.format("%s", arg.args[1]:carg())
                     end}
```
In this case, the argument will be set to the value of the first argument in the Lua function call, if not
present at call time.

### Torch Tensor types ###

`CInterface` also defines __Torch__ tensor types: `ByteTensor`,
`CharTensor`, `ShortTensor`, `IntTensor`, `LongTensor`,
`FloatTensor` and `DoubleTensor`, which corresponds to their
`THByteTensor`, etc... counterparts. All of them assume that the
[luaT](..:luaT) Tensor id (here for ByteTensor)
```
const void *torch_ByteTensor_id;
```
is defined beforehand, and properly initialized.

Additionally, if you use C-templating style which is present in the TH library, you might want
to use the `Tensor` typename, which assumes that `THTensor` is properly defined, as well as
the macro `THTensor_()` and `torch_()` (see the TH library for more details).

Another extra typename of interest is `IndexTensor`, which corresponds to a `THLongTensor` in C. Values in this
LongTensor will be incremented/decremented when going from/to C/Lua to/from Lua/C.

Tensor typenames `default` value in [wrap()](#CInterface.wrap) can take take two types:
  * A boolean. If `true`, the tensor will be initialized as empty, if not present at the Lua function call
  * A number (index). If not present at the Lua function call, the tensor will be initialized as _pointing_ to the argument at the given index (which must be a tensor of same type!).
For e.g, the list of arguments:
```lua
{
  {name=DoubleTensor, default=3},
  {name=double, default=1.0},
  {name=DoubleTensor}
}
```
The first two arguments are optional. The first one is a DoubleTensor which
will point on the last (3rd) argument if not given. The second argument
will be initialized to `1.0` if not provided.

Tensor typenames can also take an additional field `dim` (a number) which will force a dimension
check. E.g.,
```lua
{name=DoubleTensor, dim=2}
```
expect a matrix of doubles.

<a name="CInterface.userargtypes"/>
## User Types ##

Types available by default in `CInterface` might not be enough for your needs. Also, sometimes you might
need to change sliglty the behavior of existing types. In that sort of cases, you will need to
know more about what is going on under the hood.

When you do a call to [wrap()](#CInterface.wrap),
```lua
interface:wrap(
   "numel", -- the Lua name
   "numel", -- the C function name, here the same
   -- now we describe the 'arguments' of the C function
   -- (or possible returned values)
   {
      {name="DoubleTensor"},
      {name="int", creturned=true} -- this one is returned by the C function
   }
)
```
the method will examine each argument you provide. For example, let's consider:
```lua
{name="int", creturned=true}
```
Considering the argument field `name`, __wrap__ will check if the field
`interface.argtypes['int']` exists or not. If it does not exist, an error will be raised.

In order to describe what happens next, we will now denote
```lua
arg = {name="int", creturned=true}
```
First thing which is done is assigning `interface.argtypes['int']` as a metatable to `arg`:
```lua
setmetatable(arg, interface.argtypes[arg.name])
```
Then, a number of fields are populated in `arg` by __wrap__:
```lua
arg.i = 2 -- argument index (in the argument list) in the wrap() call
arg.__metatable = interface.argtypes[arg.name]
arg.args = ... -- the full list of arguments given in the wrap() call
```


[wrap()](#CInterface.wrap) will then call a several methods which are
assumed to be present in `arg` (see below for the list).  Obviously, in
most cases, methods will be found in the metatable of `arg`, that is in
`interface.argtypes[arg.name]`. However, if you need to override a method
behavior for one particular argument, this method could be defined in the
table describing the argument, when calling [wrap()](#CInterface.wrap).

The extra fields mentionned above (populated by __wrap__) can be used in the argument
methods to suit your needs (they are enough to handle most complex cases).

We will now describe methods which must be defined for each type. We will
take as example `boolean`, to make things more clear. If you want to see
more complex examples, you can have a look into the `types.lua` file,
provided by the __wrap__ package.

### helpname(arg) ###

Returns a string describing (in a human readable fashion) the name of the given arg.

Example:
```lua
function helpname(arg)
   return "boolean"
end
```

### declare(arg) ###

Returns a C code string declaring the given arg.

Example:
```lua
function declare(arg)
   return string.format("int arg%d = 0;", arg.i)
end
```

### check(arg, idx) ###

Returns a C code string checking if the value at index `idx` on the Lua stack
corresponds to the argument type. The string will appended in a `if()`, so it should
not contain a final `;`.

Example:
```lua
function check(arg, idx)
   return string.format("lua_isboolean(L, %d)", idx)
end
```

### read(arg, idx) ###

Returns a C code string converting the value a index `idx` on the Lua stack, into
the desired argument. This method will be called __only if__ the C check given by
[check()](#CInterface.arg.check) succeeded.

Example:
```lua
function read(arg, idx)
   return string.format("arg%d = lua_toboolean(L, %d);", arg.i, idx)
end
```

### init(arg) ###

Returns a C code string initializing the argument by its default
value. This method will be called __only if__ (1) `arg` has a `default`
field and (2) the C check given by [check()](#CInterface.arg.check)
failed (so the C code in [read()](#CInterface.arg.read) was not called).

Example:
```lua
function init(arg)
   local default
   if arg.default then
      default = 1
   else
      default = 0
   end
   return string.format("arg%d = %s;", arg.i, default)
end
```

### carg(arg) ###

Returns a C code string describing how to pass
the given `arg` as argument when calling the C function.

In general, it is just the C arg name itself (except if you need to pass
the argument "by address", for example).

Example:
```lua
function carg(arg)
   return string.format('arg%d', arg.i)
end
```

### creturn(arg) ###

Returns a C code string describing how get the argument if it
is returned from the C function.

In general, it is just the C arg name itself (except if you need to assign
a pointer value, for example).

```lua
function creturn(arg)
   return string.format('arg%d', arg.i)
end
```

### precall(arg) ###

Returns a C code string if you need to execute specific code related to
`arg`, before calling the C function.

For e.g., if you created an object in the calls before, you might want to
put it on the Lua stack here, such that it is garbage collected by Lua, in case
the C function call fails.

```lua
function precall(arg)
-- nothing to do here, for boolean
end
```

### postcall(arg) ###

Returns a C code string if you need to execute specific code related to
`arg`, after calling the C function. You can for e.g. push the argument
on the stack, if needed.

```lua
function postcall(arg)
   if arg.creturned or arg.returned then
      return string.format('lua_pushboolean(L, arg%d);', arg.i)
   end
end
```

