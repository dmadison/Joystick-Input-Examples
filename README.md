# Joystick Input Examples
This guide aims to contain everything you need to know about implementing joystick input in PC games. Prerequisites: familiarity with Windows programming and C-style C++.

## Table of Contents
- [Types of Devices](#types-of-devices)
	- [HID](#hid)
	- [XInput](#xinput)
- [Inputs](#inputs)
- [Outputs](#outputs)
- [APIs](#apis)
	- [Multimedia](#multimedia)
	- [DirectInput](#directinput)
	- [RawInput](#rawinput)
	- [XInput](#xinput)
	- [Files](#files)
	- [Libraries](#libraries)
		- [SDL](#sdl)
		- [Steam](#steam)
- [Specialized I/O](#specialized-io)
	- [Dualshock 4](#dualshock-4)
- [Button Configuration](#button-configuration)
	- [Controller Database](#controller-database)
	- [Calibration](#calibration)
	- [Cutting Our Losses](#cutting-our-losses)
- [Detecting Device Changes](#detecting-device-changes)
- [Displaying Physical Buttons](#displaying-physical-buttons)

## Types of devices

### HID
Standing for Human Interface Device, HID is a standardized communication protocol for USB devices a user directly interacts with, covering anything from keyboards to treadmills. An HID `usage page` is a number describing the purpose of the device, and `usages` are further subcategories. Modern game controllers use HID usage page 0x01 with usage 0x04 or 0x05, and Windows has several APIs for working with them. You can read the [official usage table from usb.org](https://usb.org/sites/default/files/hut1_21_0.pdf), or [this more concise reference](http://www.freebsddiary.org/APC/usb_hid_usages.php).

HID refers to the standard protocol, physical devices (the gamepad in your hands), and virtual devices. Games interface with the virtual device while a driver handles communicating those bytes to the physical one. I'll be redundantly referring to game controllers as HID devices.

### XInput
The only controllers you're likely to find that don't use HID are Mirosoft's own XBox controllers. These use a custom communication protocol, and Microsoft introduced a new API to work with them. We'll refer to the driver, the API, and the protocol as XInput. It supports up to 4 controllers at a time, with a light on the controller indicating the player index.

The XInput API is one of the simplest APIs Microsoft has ever made and is extremely easy to use, but it only works with XBox controllers. This is nice for indie developers as XBox controllers are a small, consistent subset of the wide world of USB game controllers, and doing XBox support well is significantly easier than supporting all possible controllers. The downside is alienating players who only have a PS4 controller, or some other controller that uses HID.

XBox controllers also work on older games that only implemented support for HID. The XInput driver creates a virtual HID device and forwards inputs to it. Effectively there are two copies of each XBox controller on your system, the XInput version and the HID version. The HID version comes with a big, annoying downside: the XInput driver combines both shoulder tiggers into one axis when converted to HID. Pressing RT makes the axis go in the negative direction, and LT makes it go in the positive direction. If you press both at the same time, it's the same as pressing neither.

So, you need XInput to get full correct input from XBox controllers, and you need HID to get input from any other controller. In order to do joystick support well, a game needs to implement support for both. You can get a list of all HID devices on your system to check which ones are XBox controllers and ignore them with the HID API, then use XInput to process them. There's unfortunately no way to associate a virtual HID device with an XInput player index.

## Inputs
The basic inputs on joysticks are
- Buttons, delivered as bitflags.
- Values for analog sticks and shoulder triggers axes, delivered as signed or unsigned integers. Analog sticks have 2 axes and triggers have one.
- A POV hat for the d-pad. This can be implemented as bitflags or an integer representing a direction, such as 1 to the north, 2 to north-east, 3 to east, etc. and 0 for being centered.

Joysticks may support non-standard inputs such as touchpads and accelerometers. For HID devices, these are delivered along with standardized data, but are specific to the device and must be reverse-engineered. See [Specialized I/O](#specialized-io) for how to access device-specific data.

## Outputs
Outputs are usually non-standard extensions, such as
- Force feedback, a.k.a. rumble or haptics
- LEDs
- Audio

Older joysticks supported HID force feedback, but not the controllers currently popular for games; XBox and PS4 controllers can't do rumble through HID. XInput provides a function to cause rumble on XBox controllers. You can set rumble and LED color on PS4 controllers by directly writing reverse-engineered data through RawInput.

Some APIs have "Effects" for force feedback that get generalized and complex, but XBox and PS4 rumble is activated through two simple linear values—one for each motor in the gamepad. If a program never turns off rumble, XBox controllers will stop rumbling when the program terminates. PS4 controllers will continue to rumble, but stop after a few seconds of receiving no instructions.

## APIs
We'll do a brief overview of each API. The `src` folder contains small example programs to illustrate implementation details. You can run them in Visual Studio by opening `Joystick Input Examples.sln` in the `vs` folder, or by running `build.bat` with a Visual Studio developer command line and launching the `.exe`s. For simplicity, most of the examples expect one controller to be connected. The `combined` example demonstrates support for any number.

### Multimedia
Multimedia Joystick is the simplest HID API for Windows. The `Ex` version of the functions and structs support up to 16 controllers at a time, each with up to 32 buttons, 6 axes, and a hat.

Getting the joystick state is one function call to `joyGetPosEx`, which takes a device index and a struct to fill with data. [Documentation for this function](https://docs.microsoft.com/en-us/windows/win32/api/joystickapi/nf-joystickapi-joygetposex) states the device index starts at zero, but on my Windows 10 machine it starts at 1.

Multimedia's functionality is quite limited. It doesn't give you any handles to hardware, so you can't tell what's an Xbox controller to use XInput instead. It also doesn't have any force feedback support.

### DirectInput
DirectInput is lower level and gives a bit more control. It uses a COM interface, so you'll be creating an object that's used to create other objects. First create a DirectInput object with `DirectInput8Create()`. You can use `IDirectInput*` or `IDirectInput8*`; the `8` versions give access to full features, though those features are not used in the `directinput` example.

To find available joysticks you'll have to enumerate them through a callback function. DirectInput will invoke this function once for each joystick plugged in to your machine.

[MSDN provides a function](https://docs.microsoft.com/en-us/windows/win32/xinput/xinput-and-directinput) to check if a device is an XBox controller while enumerating HID devices. It searches through all devices on the system to see if they have "IG_" in the device name. If it does, it compares the product and vendor IDs to the direct input device passed into the function, and, if they match, returns that the device is an XBox controller.

Device Caps, short for capabilities, contains information about what features the device has, such as number of buttons.

### RawInput
The first step with RawInput is registering to receive `WM_INPUT` events. Unlike the other APIs, RawInput uses an event queue to get inputs. This has a distinct advantage of being able to get inputs that may have been missed between frames, such as a button being quickly pressed and released (though most games don't worry about this; it would have to be running well under 30fps for players to notice). To register for events, fill out one or more `RAWINPUTDEVICE` structs and pass an array of them to RegisterRawInputDevices(). The structs take the HID usage page and usages of devices for which to receive events, bit flags, and a handle to the window whose window procedure will receive the events. You can use your game window's existing procedure, or create a non-visible window to encapsulate joystick code. The `rawinput` and `combined` examples use a non-visible window to get events in a terminal-based program.

The `lParam` for a `WM_INPUT` message is a handle to the RawInput data for that event. Each `WM_INPUT` has its own RawInput data, so if it takes longer to process an event than the update frequency of the joystick, the queue can get seriously backed up. Pass the RawInput data handle to GetRawInputData to get the actual input buffer, along with a handle to the RawInput device that can be used to get more information about it.

[Preparsed Data](https://docs.microsoft.com/en-us/windows-hardware/drivers/hid/preparsed-data) describes what kind of data is in the input buffer. We can pass the preparsed data along with the input buffer to the `HidP_*` set of functions to extract analog and digital values.

The `rawinput` example prints out all available analog values, while the `combined` example looks at usages to determine which values correspond to analog sticks and triggers on a typical HID joystick. Just like how the devices we're interested in are described by a "Joystick" (0x04) or "Game pad" (0x05) usage, each input on the device is also described by a usage. DirectInput uses X, Y, Z, Rx, Ry, Rz, Dial, and Slider (0x30-0x37), as you can see in its [DIJOYSTATE struct](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/ee416627(v=vs.85)).

The [documentation for all this on MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/_hid/) is pretty sparse and hard to understand. Open up hidpi.h for much better explanations. But I'm still not sure how the buttons and values are technically supposed to be parsed. There's a lot of strange nuance, like the return value of `HidP_GetUsageValue` is unsigned, but `LogicalMin` and `LogicalMax` of the value cap is signed. My XBox controller, through HID, says logical min is 0 and logical max is -1. I'm guessing that these were reinterpreted and sign extended from a smaller unsigned type. For the combined example, I assumed the value range is `[0, (2^bitsize)-1]`, but I don't know if this is true for all controllers.

### XInput
XInput is a simple API similar to multimedia joystick. It only works with XBox controllers, but makes getting controller state and setting rumble nice and easy.

XInputGetState() has been known to cause a several millisecond hang when trying to access non existant devices, for example, asking for player 2 input when only one controller is plugged in. You'll probably want to query which controller indices are available once, and only get regular updates from devices you know are connected. See the [Detecting Device Changes](#detecting-device-changes) section for more details.

### Files
You can interact with a device by opening it with `CreateFile()`. This gives you maximum control over the device; you just have to know what bytes to use. It's straightforward way to communicate with popular devices that are worth your time to hard code. [Read Jan Axelson's "HIDs up" article](https://www.embedded.com/hids-up/) for an introduction to low-level HID I/O. For XBox controllers, [MMozeiko has an excellent example program](https://gist.github.com/mmozeiko/b8ccc54037a5eaf35432396feabbe435) that uses file I/O instead of the XInput API, and [Dave Midison has a great walkthrough of using Wireshark](https://www.partsnotincluded.com/understanding-the-xbox-360-wired-controllers-usb-data/) to reverse engineer the data needed.

### Libraries

#### SDL
The most popular joystick libraries are SDL's Joystick and Game Controller interfaces. Joystick provides generic button, axis, and hat data. Game Controller refers to inputs in terms of an xbox controller, giving uniform semantics to all kinds of controllers (see [Controller Database section](#controller-database)).
#### Steam
The [Steam API provides joystick support](https://partner.steamgames.com/doc/api/isteaminput) with action mapping, rumble, and LED control, among other features. It's probably best to use this API when shipping your game on Steam for consistent user experience (though you'll still need other options if your game is available elsewhere).

## Specialized I/O
In addition to XBox controllers, Playstation 4 and Nintendo Switch controllers are popular for PC games at time of writing. The basic functionality of these controllers is HID compliant and will work with any of the HID APIs, but you'll need specialized code if you want to make use of their non-standard features. You can check the product and vendor IDs of a device against a list of known devices to determine the type of controller.

The `specialized` example shows how to implement full support for Playstation 4 controllers. It starts similarly to the `rawinput` example, but instead of passing raw data to `HidP_Get*` functions, it interprets the data directly.

In addition to specialized parsing of input, we can control LEDs and rumble using `WriteFile` or `HidD_SetOutputReport`. These two functions use slightly different types of communication on the backend, and you'll need to use the right one for the device. Getting input is fast (around 0.02ms on my machine), but sending output can be slow (up to 10ms on my machine). Therefore each controller will need a dedicated output thread.

### Dualshock 4
A Playstation 4 controller can be connected by a USB cable or bluetooth. For the most part they use the same data, but bluetooth has an extra two bytes at the beginning of input and output reports. While [Playstations's own website](https://www.playstation.com/en-us/support/hardware/ps4-pair-dualshock-4-wireless-with-pc-or-mac/) states that Dualshock 4 controllers don't support rumble and changing LED color on PC when connected by bluetooth, it is in fact possible, you just have to use the right type of output. Use `WriteFile` for USB and `HidD_SetOutputReport` for bluetooth.

Use the size of the input report to determine if a dualshock 4 is wired or bluetooth. the first byte of a wireless report should be 0x11, but may be different if it was misconfigured by your application or another. For example, if WriteFile is used for bluetooth, the controller will then send packets similar to wired mode, but only containing standard HID data.

## Button configuration
Ideally, players should have full control over how their controller inputs map to game actions through an intuitive interface. Some players will want to play your retro-style game with a Super Nintendo controller going through a random SNES-to-USB converter they found on ebay. Other players have physical disabilities and need to use unusual button mappings or custom-built controllers. Button config is crucial for player experience, but presents numerous complications for developers.

The gold-standard for button config is press-to-set. The player selects a game action, then presses the corresponding button to map it. Even better is if the next action is automatically selected, so the player can quickly map all the buttons in a row. Directional inputs can configurable as well by separating an analog stick into -x, +x, -y, and +y. So to do full button config you need to iterate through all inputs on the controller: check if a button is pressed, if an axis passed a threshold, or if the hat is not centered. The `Combined` example puts all inputs into an array of floats, each in the range `[0,1]` (buttons and cardinal hat directions are 0 or 1 while analog inputs can be in between). This makes button config code simple as the inputs are generic.

The first problem with this kind of button config is that axes are not always centered at rest. Shoulder triggers on XBox controllers are 0 at rest, 255 when fully pressed. If these were mapped to HID just like any other axis, it would appear to a game that those axes are always being pushed in the negative direction. This is [Microsoft's rationale](https://docs.microsoft.com/en-us/windows/win32/xinput/xinput-and-directinput) for mapping both triggers to the same axis for HID. The correct implementation would have been to map them to separate axes that are 0 at rest, and forget about the negative range, but we're stuck with this decision. If you want to support as many controllers as possible, you have to deal with this anyway: PS4 and Gamecube controllers, for example, report the shoudler triggers as negative at rest.

PS4 controllers have another problem: R2 and L2 report both a button and an axis. This could lead to the being mistakenly mapped to two actions, or the button being mapped where an analog value is wanted. The `Combined` example throws out the button to avoid this problem, as well as mapping PS4 and XBox shoulder triggers to a `[0,1]` range.

So what of generic HID devices? How do we correctly map inputs when they could be anything?

### Controller Database
SDL's Game Controller interface brings the idea of expressing every controller one could possibly plug into a PC in terms of an XBox controller. They crowd-sourced a database that maps hardware IDs to button configurations, eliminating the headaches of generic input. You can then apply your game's own mapping, and effectively only worry about xbox controllers while supporting many more than that. Another advantage is more accurate names to display to users; the Playstation 4 controller calls itself "Wireless Controller", but a database can call it "PS4 Controller" or "Dualshock 4."

SDL includes the database in its source code and is ready to go if you use the Game Controller interface. You can also download the database and parse it yourself if you're not using SDL: https://github.com/gabomdq/SDL_GameControllerDB

The one problem with the database approach is that it doesn't work with USB converters that support multiple types of controllers, as they all share one hardware ID. For example, I have a converter which supports PS2, Gamecube, and original XBox controllers: https://www.mayflash.com/Products/UNIVERSAL/PC035.html
Somebody at some point made an entry in the SDL database for this device, and mapped it for gamecube controllers. I use it for a PS2 controller, so the mapping is totally wrong and renders some buttons unusable. Still, the database approach will work fantastically well for nearly all of your players.

### Calibration
I remember many times hastily plugging a controller into a Gamecube, bumping an analog stick while doing so, causing it to drift. Pushing it to the left made the cursor on character select drift to the right; pushing it up caused drifting downward. This is because the gamecube calibrates the controller when it's plugged in. Whatever the analog stick's current axis values were are now considered the rest position, and all inputs are relative to that. If an axis is calibrated for a negative value being the rest position, the axis' physical neutral will appear to be a positive input. Unplugging and reconnecting the controller—or holding down X, Y, and start—recalibrates it.

A PC game could also use calibration to account for unknown rest positions of axes. By sampling input state and taking it as the rest values when starting button config, a trigger that's `LONG_MIN` at rest would only appear to go in the positive direction. The downside is potential player confusion if they were bumping an analog stick when starting button config.

### Cutting our losses
Really, we can still get decent button config without a database or calibration. Let's imagine a player configuring their Gamecube controller in a press-to-set interface. They want to map the Shoot action to the right shoulder trigger, so they select the action and press down R. The input was negative both this frame and last, so we ignore the initial state. It isn't until the input becomes positive that we create the mapping. After Shoot is Jump, and when the player releases R they accidently map Jump to a negative axis input. They curse, redo that mapping, and move on. So while not perfect, it does support an uncommon controller and lets the user create any mappings they want.

## Detecting Device Changes
Players may not have all their controllers connected when they launch your game. A surprising number of PC games enumerate controllers only when they start, requiring a full restart of the program to detect any devices that were plugged in while the game was running.

Enumeration can be slow—I've seen DirectInput take over 70ms to complete enumeration, so doing it every frame isn't an option. `SDL_NumJoysticks()` seems to only re-enumerate every few seconds, so it runs smoothly then causes a massive hitch if called every frame. So you should only enumerate when you're sure devices have been changed. It might still cause a frame drop, but the player is unlikely to notice while they're fiddling with USB ports.

Hotplugging, as it is often called, is easy to support at the Windows API level. For RawInput, use the `RIDEV_DEVNOTIFY` flag when registering for events, and the window procedure will receive `WM_INPUT_DEVICE_CHANGE` messages whenever a device is plugged in or removed. A handle to that device will be available in `lParam`. For other APIs, add the following during program startup:
```C
DEV_BROADCAST_DEVICEINTERFACE notificationFilter = { sizeof(DEV_BROADCAST_DEVICEINTERFACE), DBT_DEVTYP_DEVICEINTERFACE };
RegisterDeviceNotification(hwnd, &notificationFilter, DEVICE_NOTIFY_WINDOW_HANDLE);
```
This will register your window procedure to receive `WM_DEVICECHANGE` messages. These will also be sent when a device is plugged in or taken out. SDL offers similar events, `SDL_JOYDEVICEADDED` and `SDL_JOYDEVICEREMOVED`. Once you know there's been a change in available devices, you can re-enumerate them.

Handling hot-plugging at the game-logic level is probably going to be a lot more complicated. You may want to notify the player and/or pause the game when their controller is unplugged. Charging cables that come with PS4 controllers are notoriously loose, and *Cuphead* saved me many times by pausing the game when my controller disconnected.

A multiplayer game could have several controllers plugged in and removed frequently as players join in and take breaks. If you just associate a player with index into the array of all available joysticks, player 3 could suddenly become player 2 when a joystick is disconnected. Even worse, button configurations specific to each device or player could become mixed up. The XInput driver handles this well for XBox controllers: player 2 will still be player 2 even if player 1 disconnects. For HID, you'll need some unique ID for each joystick to keep track of them. The device handle changes when a controller is disconnected and reconnected, so the `combined` example uses `RIDI_DEVICENAME` from RawInput. It keeps disconnected joysticks around in the array so it can put them back in the same place if reconnected, and user code would only have to worry about index for player and button config associations.

## Displaying Physical Buttons
Console games have long displayed icons instead of words to tell players which button to press. Modern PC games are also using button icons, a.k.a. glyphs, in their displayed text for popular controllers. Steam provides a set of glyphs for games to use that's accessible [through their API](https://partner.steamgames.com/doc/api/isteaminput#GetGlyphForActionOrigin). Games like *Hades* have their own set of glyphs to match the game's aesthetic.

Strings displayed to the user will need to encode logical games actions for which a glyph will be displayed; for example, `"Press [action:jump] to jump!"`. Before displaying the string, ask the button configuration which physical button maps to that action, and use that as a key to look up the glyph. The `combined` example has unique strings for each physical button that could be used as keys in a glyph table.