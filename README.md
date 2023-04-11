# Synchronizing data via a selection event
This library is designed to quickly synchronize data between players. The transfer occurs through calls to the local click event on the unit.

## Installation

To install, copy the UnitSync folder, copy dummy from the Object Editor, make sure that the dummy code matches the UNIT_ID value in the USync trigger. Use. There are also two examples of periodic data synchronization in the map, passing integer and real.

## Integer transfer
```scala
function event_kill_count takes nothing returns nothing
    local integer data = USync.integer
    local integer player_id = USync.player_id
    call BJDebugMsg("sync kill_count"+I2S(data))
    call BJDebugMsg("sync player_id"+R2S(player_id))
endfunction

function send_my_value takes nothing returns nothing
  if GetLocalPlayer() == Player(0) then
     call USync.send("kill_count", 37)
  endif
endfunction

function trigger_init takes nothing returns nothing
    call USync.register_integer("kill_count", 0, 1000, true, function event_kill_count)
endfunction
```
In this example, we first register the message using Rsync.register_integer() where we pass the callback function that will be called after receiving the data.

## Real transfer
```scala
function event_damage_multiplier takes nothing returns nothing
    local real data = USync.real
    call BJDebugMsg("sync damage_multiplier"+R2S(data))
endfunction

function send_my_value takes nothing returns nothing
  if GetLocalPlayer() == Player(0) then
     call USync.send("damage_multiplier", 15.)
  endif
endfunction

function trigger_init takes nothing returns nothing
    call USync.register_real("damage_multiplier", -50, 50, 0.1, true, function event_damage_multiplier)
endfunction
```
Real numbers always come with an error because we don't transmit all 32 bits. In this example, 0.1 is the maximum allowable error. For example, if you send 15.0, then expect that a number from 14.9 to 15.1 can come.

## Transmitting coordinates
https://user-images.githubusercontent.com/103655830/231171799-7937ba38-1f88-4d1f-8c9d-a38f016566ce.mp4
```scala
function send_position takes nothing returns nothing
    local real x = GetCameraTargetPositionX()
    local real y = GetCameraTargetPositionY()
    
    if GetLocalPlayer() == Player(0) then
        call USync.send_coordinate("unit_position", x, y)
    endif
endfunction

function event_get_position takes nothing returns nothing
    local real x = USync.x
    local real y = USync.y
    call SetUnitX(UNIT, x)
    call SetUnitY(UNIT, y)
    call send_position()
endfunction

function trigger_init takes nothing returns nothing
    call USync.register_coordinate("unit_position", 1, true, function event_get_position )
endfunction
```
The Sync.register_coordinate() function is intended for registering events of sending data whose type is the coordinates of the game map plane. In the example, we synchronize the coordinates of the local player's camera and move the unit there.

## Specifications
```
method register_real takes string name, real min, real max, real maximum_error, boolean buffering, code callback returns integer
```
Register a message. Returns the number of clicks required to send data.
- `name` - the name used to create/send a message. Re-registration will display an error message.
- `min` - the minimum possible transmitted value. Passing a value less than min will return an error message.
- `max` - the maximum possible transmitted value. Passing a value greater than max will return an error message.
- `maximum_error` - the maximum possible error during transmission. It usually takes the values [1, 0.1, 0.01]. Transmitting large ranges of values with high accuracy can be difficult.
- `buffering` - event buffering, by default true, more details below.
- `callback` - a function that will be called after data synchronization.
```
method register_integer takes string name, integer min, integer max, boolean buffering, code callback returns integer
```
Register a message. Returns the number of clicks required to send data.
- `name` - the name used to create/send a message. Re-registration will display an error message.
- `min` - the minimum possible transmitted value. Passing a value less than min will return an error message.
- `max` - the maximum possible transmitted value. Passing a value greater than max will return an error message.
- `buffering` - event buffering, by default true, more details below.
- `callback` - a function that will be called after data synchronization.

PS The `maximum_error` value is set automatically, for integer == 1.
```
method register_coordinate takes string name, real maximum_error, boolean buffering, code callback returns integer
```
Register a message. Returns the number of clicks required to send data.
- `name` - the name used to create/send a message. Re-registration will display an error message.
- `maximum_error` - the maximum possible error during transmission. It usually takes the values [1, 0.1, 0.01]. Transmitting large ranges of values with high accuracy can be difficult.
- `buffering` - event buffering, by default true, more details below.
- `callback` - a function that will be called after data synchronization.

PS The `min` and `max` values are automatically set equal to the map boundaries.
```
method send takes string name, real value returns boolean
```
Send data on a saved message. Returns a boolean success value.
- `name` - the name of the message that was specified during registration.
- `value` - the transmitted value itself, the payload.
```
method send_coordinate takes string name, real x, real y returns boolean
```
Send data on a saved message. Returns a boolean success value.
- `name` - the name of the message that was specified during registration.
- `x` - value on the x axis.
- `y` - value on the y axis.

## Buffering
To understand how buffering works, consider the following code
```scala
function test_buffering takes boolean buffering returns nothing
    call USync.register_integer("data", 0, 10, buffering , function event_callback)
    if GetLocalPlayer() == Player(0) then
      call USync.send("data", 1)
      call USync.send("data", 2)
      call USync.send("data", 3)
      call USync.send("data", 4)
      call USync.send("data", 5)
    endif
endfunction
```
If buffering == true, then five events with data 1, 2, 3, 4, 5 will be triggered after a while.

If buffering == false, then two events with values 1, 5 will be triggered after a while, since attempts to send both 2 and 3 and 4 and 5 were at the time when 1 was not synchronized yet, and after synchronization the last requested update was sent.

Where is it needed? For example, if you decide to send messages faster than they will arrive - every 0.15seconds - then messages will accumulate in a queue that will grow infinitely, and because of this, the time from the moment the event is sent to the moment it is synchronized will also begin to grow infinitely. Perhaps you are synchronizing the mouse coordinate or so, and it is important for you to get the most up-to-date data as quickly as possible, and it does not matter at all what coordinates the mouse was in before. One way to solve the problem of accumulating a queue of data receipt events is to set buffering == false. In this case, all new attempts to send data will only update the "next send", which will occur only after the last sent data is synchronized.

The second way to get rid of the accumulation of a queue of data receipt events is not to create such queues. You can update data not only with the help of a periodic timer, but, for example, send data only after receiving the previous ones. In this case, you will maintain the maximum possible data transfer rate.
