# Timing and Event loop
Luanti's Lua API can be used to get time and date or to schedule "events" to run once or every server step.

## Settings

* ***On servers***
	* The `dedicated_server_step` setting controls the time between server steps. This is the maximum granularity (the most time spent between steps and the finest timing possible) of each game loop without a busywait.

* ***In singleplayer***
	* Framerate determines server steps.

## Lua builtins
Not restricted by mod security, these functions are available to both SSMs and CSMs and allow getting times / dates.

* [`os.clock`](https://www.lua.org/manual/5.1/manual.html#pdf-os.clock) - Approximate seconds CPU time used.
* [`os.time`](https://www.lua.org/manual/5.1/manual.html#pdf-os.time) - The current time. Will usually be a UNIX timestamp in seconds.
* [`os.difftime`](https://www.lua.org/manual/5.1/manual.html#pdf-os.difftime) - Difference between two `os.time` timestamps (usually simply `b - a`).
* [`os.date`](https://www.lua.org/manual/5.1/manual.html#pdf-os.date) - Getting & formatting dates.

### Example

You can use `os.clock` for accurate benchmarks of CPU time spent:

```lua
local function slow_sum(n)
	local sum = 0
	for i = 1, n do
		sum = sum + i
	end
	return sum
end

local function benchmark(calls, func, ...)
	local time = os.clock()
	for _ = 1, calls do
		func(...)
	end
	return (os.clock() - time) / calls -- seconds of CPU time spent per call
end

print("slow_sum(1e6) takes", benchmark(1e3, slow_sum, 1e6), "seconds per call")
```

## Functions

### `core.get_us_time`

#### Usage

```lua
time = core.get_us_time()
```

#### Returns
- `time` - `{type-number}`: System-dependent timestamp in microseconds since an arbitrary starting point (µs)

{{< notice tip >}}
Divide by `1e6` to convert `time` into seconds.
{{< /notice >}}

{{< notice info >}}
The returned `time` is not portable and not relative to any specific point in time across restarts - keep it only in memory for use while the game is running.
{{< /notice  >}}

{{< notice tip >}}
You can use the difference between ``core.get_us_time`` and the returned times to check whether a real-world timespan has passed, which is useful for rate limiting. For in-game timers, you might prefer adding up `dtime` or (if second precision is enough) using gametime.
{{< /notice >}}

#### Example

It is possible to use a simple while-loop to wait for a timespan smaller than the server step. This is called a "busywait".

```lua
local start = core.get_us_time()
repeat until core.get_us_time() - start > 1e3 -- Wait for 1000 µs to pass
```

{{< notice warning >}}
This blocks the server thread, possibly delaying the sending of packets that are sent each step and creating "lag". Use this only if absolutely necessary and only for very small timespans.
{{< /notice >}}

### `core.get_gametime`

#### Usage

```lua
time = core.get_gametime()
```

#### Returns
- `time` - `{type-number}`: Time passed "in-game" since world creation in seconds ("gametime"). Does not increase while the game is paused or not running. Gametime is stored with the world and continuously increases.

#### Example

You can achieve higher precision by implementing gametime yourself, adding up `dtime`. The only tricky part about this is reading the initial gametime, which is only available at runtime and not at load time; the entire code may only run the the first server step:

```lua
myapi = {}
core.after(0, function()
	local gametime = core.get_gametime()
	core.register_globalstep(function(dtime)
		gametime = gametime + dtime
	end)
	function myapi.get_precise_gametime()
		return gametime
	end
end)
```
{{< notice info >}}
This naive implementation might be one server step ahead or behind `core.get_gametime`. Note that the initial gametime is rounded. Do not persist the values for this reason.
{{< /notice >}}

{{< notice tip >}}
Use this implementation only for measuring in-game timespans.
{{< /notice >}}

### `core.register_globalstep`

Calls the given callback every server step.

#### Usage

```lua
core.register_globalstep(function(dtime)
	-- do something, probably involving dtime
end)
```

#### Params
- `dtime` - `{type-number}`: Delta (elapsed) time since last step in seconds

{{< notice tip >}}
Use globalsteps to poll for an event for which there are no callbacks, such as player controls (`player:get_player_control()`).
{{< /notice >}}

{{< notice info >}}
As globalsteps run every server step, they are highly performance-critical and must be well optimized. If globalsteps take longer than the server step to complete, the server thread is blocked and the server becomes "laggy".
{{< /notice >}}

#### Examples

Globalsteps are ideal to run something periodically, which is often used to call expensive operations less frequently in order to keep server thread blockage - and thus lag - to a minimum.

```lua
-- period is in seconds
local function run_periodically(period, func)
	local timer = 0
	core.register_globalstep(function(dtime)
		timer = timer + dtime
		if timer > period then
			func()
			timer = 0
		end
	end)
end
run_periodically(5, function()
	core.chat_send_all("5 seconds have passed!")
end)
```

This will call `func` after at least `timer` seconds have passed. Note that there is no "catchup": The timer is always reset to zero, no matter how "late" the call to func is - how large `timer - period` is. For catchup, you can simply set the timer to the leftover time instead: `timer = timer - period`. This will trigger calls in subsequent steps until the missed time is "caught up".

### `core.after(time, func, ...)`

Will call `func(...)` after at least `time` seconds have passed.

{{< notice tip >}}
Use `core.after(0, func)` to immediately do load-time stuff that is only possible at run-time, or to schedule something for the next server step.
{{< /notice >}}

{{< notice info >}}
Scheduled calls that run in the same server step are executed in no particular order.
{{< /notice >}}

The following snippet emulates a globalstep, sending `Hello World!` to chat every server step:

```lua
local function after_step()
	core.chat_send_all("Hello World!")
	core.after(0, after_step)
end
core.after(0, after_step)
```

#### Usage
```lua
job = core.after(time, func, ...)
if ... then
	job:cancel()
end
```

#### Params
- `time` - `{type-number}`: Time in seconds that must have passed for the callback to be executed.
- `func` - `{type-function}`: Function to be called. Objects with the `__call` metamethod are supported as well.
- `...` - vararg: Arguments to be passed to `func` when it is called.

#### Returns
- `job` - job object: Simple object providing a `job:cancel()` method to cancel a scheduled "job".

{{< notice info >}}
`...` may be arbitrarily cut off at `nil` values, as Luanti uses a simple list to store the arguments. Don't include `nil`s in the arguments if possible or you may lose arguments.
{{< /notice >}}

{{< notice tip >}}
If you have to call a function with ``nil``s in it's argument list, use a closure for reliable behavior:

```lua
core.after(time, function()
	func(nil, arg, arg2, nil, nil)
end)
```
{{< /notice >}}

{{< notice note >}}
All scheduled callbacks are stored in a list until they are called. This list is traversed in linear time if *any* of the callbacks are executed. Excessive use of `core.after` may result in slow execution time.
{{< /notice >}}

## Entities

### `entity:on_step(dtime, ...)`

Callback. Called per-entity every globalstep with the current `dtime`, allowing for comfortable per-entity timing.

#### Usage
```lua
function entity:on_step(dtime, ...)
	-- use dtime
end
```

#### Params
- `self` - entity: The entity itself (implicit parameter)
- `dtime` - `{type-number}`: Delta (passed) time since last step in seconds (same as globalstep)
- `...` - vararg: Other parameters irrelevant to timing (such as `moveresult`)
