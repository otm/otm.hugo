+++
date = "2015-07-22T16:18:30+02:00"
draft = false
title = "Embedding Lua in Go"
type = "post"
+++

***Command line parameters, environment variables, and configuration files are common ways to change the behavior of software. However, sometimes that is just not enough and an embedded language can be the solution. In this case we will embed [Lua](http://www.lua.org/) using [gopher-lua](https://github.com/yuin/gopher-lua).***

*Godoc: http://godoc.org/github.com/yuin/gopher-lua*

*All code in the post can be found at http://github.com/otm/embedding-lua-in-go*

### Running Lua Code in Go
First lets set up the environment and test that it works, start by install gopher-lua:

~~~
go get github.com/yuin/gopher-lua
~~~

Secondly let's create a minimal implementation:

{{< code label="hello.go" >}}
package main

import "github.com/yuin/gopher-lua"

func main() {
	L := lua.NewState()
	defer L.Close()
	if err := L.DoString(`print("Hello World")`); err != nil {
		panic(err)
	}
}
{{< /code >}}

`lua.NewState()` creates our Lua VM, and it is though `L` *(*lua.LState)* we will interact with Lua in the future. Throughout the post `L` will denote a pointer to `lua.LState`. `L.DoString` runs the Lua code in the VM. Running the Go code will yield:
~~~ bash
$ go run hello.go
Hello World
~~~

To run a Lua file, instead of a string, call `lua.DoFile`

``` go
L := lua.NewState()
defer L.Close()
if err := L.DoFile("hello.lua"); err != nil {
		panic(err)
}
```

### Embedding Lua Code as Strings
`DoFile` and `DoString` can be called several times, and thus it can be utilized to expose Lua function. In the example bellow `sayHello` function is first defined, and then called in the second call to `DoString`

{{< code label="hello2.go" >}}
func main() {
	L := lua.NewState()
	defer L.Close()
	if err := L.DoString(`function sayHello() print("Hello Again") end`); err != nil {
		panic(err)
	}

	if err := L.DoString(`sayHello()`); err != nil {
		panic(err)
	}
}
{{< /code >}}

### Calling Go Code from Lua
Exposing Go functions to Lua is essential to create custom a custom API. A Go function should implement the `LGFunction` type to be callable from Lua.
~~~ go
type LGFunction func(*LState) int
~~~

It receives a `*lua.LState` and returns an integer. The `LState` is needed for interacting with the Lua VM, most commonly for retrieving function arguments. The returned integer defines how many return values has been pushed to the Lua stack. A complete example looks like this:

{{< code label="calling-go.go" >}}
func square(L *lua.LState) int {  //*
	i := L.ToInt(1)          // get first (1) function argument and convert to int
	ln := lua.LNumber(i * i) // make calculation and cast to LNumber
	L.Push(ln)               // Push it to the stack
	return 1                 // Notify that we pushed one value to the stack
}

func main() {
	L := lua.NewState()
	defer L.Close()

	L.SetGlobal("square", L.NewFunction(square)) // Register our function in Lua
	if err := L.DoString(`print("4 * 4 = " .. square(4))`); err != nil {
		panic(err)
	}
}
{{< /code >}}

The LState defines some convenience functions, in the example above we are using `L.ToInt(1)` to fetch the first function argument.

***Note:*** *Lua is zero-indexed, so the first function argument is fetched with `L.ToInt(1)`, second argument with `L.ToInt(2)`. In Lua, all arrays are 1-indexed, however t[0] is still valid but that would result in the lenght of the array to be off-by-one.*

There are a number of `To...(n int)` functions available. These functions does not throw errors, but will return Go default values if conversion is not possible. To get automatic errors the `L.Check...(n int)` family of functions can be used; They throw a Lua error if the type check fails. For optional arguments the `L.Opt...(n int, default T)` functions can be used. Example:

`L.GetTop()` returns the number of arguments that was used when the function was called. And to fetch an argument without conversion the `L.Get(n int)` function can be used.

If an argument to a function can be of more then one type the `L.CheckTypes(n int, types ...LValueType)` function can be used to check and yield an error to the user. Using the `L.CheckTypes` function equate to checking the type manually and then calling `L.TypeError(n int, message string)` if there is an error.

``` go
// Second argument can be string or function
L.CheckTypes(2, lua.LTString, lua.LTFunction)
switch lv := L.Get(2).(type) {
case LString:
	// use as string
case Lfunction:
  // use as function
}
```

### Calling Lua from Go
Calling Lua code is done through `L.CallByParam`, which takes a parameters object, [P](http://godoc.org/github.com/yuin/gopher-lua#P), and arguments as variadic parameters. The parameters object takes three important parameters:

 * Fn - the lua.LFunction to call
 * Nret - The number of returned values
 * Protect - If protect is `true` an error is returned, otherwise a panic will occur.

The following code defines a "concat" function in Lua. Calls the concat function with the arguments "Go" and "Lua" and prints the resulting string to stdout.

{{< code label="calling-lua.go" >}}
// luaCode is the Lua code we want to call from Go
var luaCode = `
function concat(a, b)
	return a .. " + " .. b
end
`

func main() {
	L := lua.NewState()
	defer L.Close()

	if err := L.DoString(luaCode); err != nil {
		panic(err)
	}

	// Call the Lua function concat
	// Lua analogue:
	//	str = concat("Go", "Lua")
	//	print(str)
	if err := L.CallByParam(lua.P{
		Fn:      L.GetGlobal("concat"), // name of Lua function
		NRet:    1,                     // number of returned values
		Protect: true,                  // return err or panic
	}, lua.LString("Go"), lua.LString("Lua")); err != nil {
		panic(err)
	}

	// Get the returned value from the stack and cast it to a lua.LString
	if str, ok := L.Get(-1).(lua.LString); ok {
		fmt.Println(str)
	}

	// Pop the returned value from the stack
	L.Pop(1)
}
{{< /code >}}

### Good to Know

#### Gopher-Lua Types
The gopher-lua library operates on a interface called `LValue`.
``` go
type LValue interface {
  String() string
  Type()   LValueType
}
```
Objects implementing this interface are `LNilType`, `LBool`, `LNumber`, `LString`, `LFunction`, `LUserData`, `LState`, `LTable`, and `LChannel`. Calling the `LValue.Type()` returns the corresponding LValueType.

``` go
const (
  LTNil LValueType = iota
  LTBool
  LTNumber
  LTString
  LTFunction
  LTUserData
  LTThread
  LTTable
  LTChannel
)
```

### Covertion and Check functions
There are some practical functions to convert and check `lua.LValue` objects.

  * `lua.LVAsBool(v LValue)` - Convert to bool, nil and false becomes false. Otherwise true.

  * `lua.LVAsString(v LValue)` - Converts LString and LNumber to string. Otherwise empty string.
  * `lua.CanConvToString(v LValue)` - True if LString or LNumber. Otherwise false.
  * `lua.LVIsFalse(v LValue)` - Returns true if nil or false.

### The LTable type
One of the most versatile and used data structures in Lua is LTable (actually it is the only one). The LTable type can be used for emulating namespaces and classes. However, the basic API is quite simple, and the advanced features deserves its own post. See http://godoc.org/github.com/yuin/gopher-lua#LTable for the LTable Go API.
