---
layout: post
title:  "Translating zig types to dbus"
date:   2026-05-31 19:42:27 -0600
categories: jekyll
---
*This post documents building part of the [wayland widgets](https://github.com/sesh9096/wayland-widgets) project and all code can be found there*

# Introduction
I first got started running linux on my personal computers about 3 years ago and have been using wayland for graphics particularly the river compositor for the last year.
On the whole, I quite liked teh experience of making my computing experience customized for doubtful purposes, but a small part of my brain kept nagging at me that I really had no more idea of how most of what I was doing worked than when I was on ~~Windows~~..
So one day, I resolved to change that and have regretted it ever since.
My end goal with this project is to create a functional status bar, notification manager, wallpaper manager, and all other portions of the desktop shell I would find useful without using any preexisting widget toolkit.
I choose Zig as my language for this project because I wanted to try something new and had liked the power of C but found myself impeded by things such as the lack of generics and awkward meta-programming.

At the time of this posting, I have managed to get some very basic displays and statuses, but for some very crucial components such as system tray icons and notifications, I would need to figure out dbus.

## A short introduction to dbus
Dbus is a messaging protocol on linux which allows applications to send standardized messages via a central bus process rather than to each process individually.
On desktop linux, dbus is the preferred method for sending notifications, system tray icons, accessing system resources via portals, and many other things besides without an application needing to know the particular implementation.

The general structure of communication can be described as follows
- After connecting to the bus, a client may register any number of names by which it wants to be known.
  These can include well known names for certain services and are conventionally in reversed DNS form such as `org.freedesktop.Notifications`.
  Messages sent to that bus name will then be routed by the bus to that client.
- A client may then register objects which are identified by a path such as `/org/freedesktop/Notifications`.
- Each object implements interfaces which have names which generally have the same form as bus names.
- Interfaces have methods which can be called and can emit signals which can be listened for.

Is this perhaps overcomplicated and/or OOP centric? Quite possibly, but it seems to work and is now essential for a lot of application functions.

There are several implementations of the dbus protocol, but for the purposes of this project, we will be using `libdbus` a low level C library which provides both the reference/main bus implementation and a client library to connect to and communicate with this bus.
However, this library is very low level (missing many convenient features), so we will have to implement a lot of functionality to create something for our application to use.

## A short introduction to zig
[Zig](ziglang.org) could be described as a modern language somewhat in the style of C with features such as compile time metaprogramming and type reflection which will be very important as will be shown later.
If you were already interested and this post has not dissuaded you, I would highly recommend checking it out.

Apart from the features already mentioned, I choose zig for its claimed C interoperability, low level controls, explicit error handling, and other nice modern features as well as its simplicity.
This project is written in version 0.13 and is likely to break in subsequent versions since zig is still very much unstable.

# Implementation
**Warning**: Do not expect good or functional code, this project is still very much a WIP without even a clear direction for where it is going.

To send messages over dbus, I first had to figure out more or less how it worked:
mostly from the [specification](https://dbus.freedesktop.org/doc/dbus-specification.html) and `libdbus` [documentation](https://dbus.freedesktop.org/doc/api/html/),
For this project, we should ideally be able to send and receive a variety of messages,
but a good start would be sending a notification.

## Setup
On my linux distro of choice(Void Linux), libdbus is provided as part of the `dbus` package and the header files in `dbus-devel`

## Connection to the bus
The dbus protocol specifies 2 main use cases: a "system bus" for a system i.e. a daemon started perhaps on initialization and "session bus" usually associated with the graphical session.
To connect to the session bus we could write the following in C:
```c
#include <dbus/dbus.h>
#include <stdio.h>

int main(int argc ,char** argv){
  DBusConnection* connection = dbus_bus_get(DBUS_BUS_SESSION, NULL);
  if(connection) printf("Connected to session bus");
}
```

Doing this in zig turned out to be simpler than expected.
Zig provides an inbuilt feature to translate c code into zig code which allows us to write the following:

```zig
const std = @import("std")
const c = @cImport({
    @cInclude("dbus/dbus.h");
});

pub fn main() !void{
    const bus = c.dbus_bus_get(c.DBUS_BUS_SESSION, null);
    if(bus != null) std.debug.print("Connected to session bus", .{});
}
```
Then, the harder part is linking with libdbus which can be done in build.zig with `dbus.linkSystemLibrary("dbus-1", .{});` or by adding `-ldbus-1` on the command line.
Luckily, zig uses pkgconfig when system libraries, so the headers get included.

## Working with Messages
`libdbus` provides several methods to create messages, what we need is `dbus_message_new_method_call` since we intend to call the `Notify` method on the relevant interface.

We could continue using the automatically generated code produced by zig's automatic c translation, but to take advantage of some of zig's features such as more pointer types and namespaces, we need to write some bindings.
As an example:
```zig
pub const Message = opaque {
    // ...

    pub extern fn dbus_message_new_method_call(destination: ?[*:0]const u8, path: [*:0]const u8, iface: ?[*:0]const u8, method: [*:0]const u8) ?*Message;
    pub const newMethodCall = dbus_message_new_method_call;

    // ...
};
pub fn main() !void{
    const message = Message.newMethodCall("org.freedesktop.Notifications", "/org/freedesktop/Notifications", "org.freedesktop.Notifications", "Notify").?;
}
```

Adding arguments to a message is a bit trickier, the libdbus documentation mentions using either the `dbus_message_append_args` method or creating `DBusMessageIter` to append arguments.
I choose to exclusively use `DBusMessageIter` because it is more versatile, and, while being harder to set up, would be needed anyways for appending container type arguments like arrays and variants.

Since the goal was to create something which made sense in zig, my idea was to follow the example of zig format and allow arguments to be submitted as a struct which would also solve encoding type information.
The end result allows for something like this:

```zig
pub fn main() !void{
    const message = Message.newMethodCall("org.freedesktop.Notifications", "/org/freedesktop/Notifications", "org.freedesktop.Notifications", "Notify").?;

    // fn (message: *Message, args: anytype) Allocator.Error!void
    message.appendArgsAnytype(struct {
        app_name: [*:0]const u8 = "",
        replaces_id: u32 = 0,
        app_icon: [*:0]const u8 = "",
        summary: [*:0]const u8 = "Hello world",
        body: [*:0]const u8 = "",
        actions: []const [*:0]const u8 = &.{},
        hints: dbus.Vardict = &.{},
        expire_timeout: i32 = -1,
    } {});
}
```

The `appendArgsAnytype` relies on zig's type reflection with the `@TypeOf` and `@typeInfo` functions which allows it to call the correct underlying libdbus message iterator functions with a reasonable type.
It also means we can use recursion to append things like arrays and variants(which are a whole headache to implement but not relevant here).
See this [link](https://github.com/sesh9096/wayland-widgets/blob/3a7a127af3f6ac5b8851e2c7bc4474285f5220c4/src/dbus/dbus.zig#L566) for more details.

Now that we have a message, we need to send it.

## Sending

There are several ways to send messages on dbus, but for simplicity, we will use `dbus_connection_send_with_reply_and_block`, so the full code to send a notification will look something like this:
```zig
// see github
const dbus = @import("dbus");
pub fn main() !void{
    const dbus_connection = dbus.busGet(.session, null).?;
    const message = dbus.Message.newMethodCall("org.freedesktop.Notifications", "/org/freedesktop/Notifications", "org.freedesktop.Notifications", "Notify").?;

    // fn (message: *Message, args: anytype) Allocator.Error!void
    message.appendArgsAnytype(struct {
        app_name: [*:0]const u8 = "",
        replaces_id: u32 = 0,
        app_icon: [*:0]const u8 = "",
        summary: [*:0]const u8 = "Hello world",
        body: [*:0]const u8 = "",
        actions: []const [*:0]const u8 = &.{},
        hints: dbus.Vardict = &.{},
        expire_timeout: i32 = -1,
    } {});
    const reply = dbus_connection.sendWithReplyAndBlock(message, 10000, null).?;
    reply.unref();
}
```

# Conclusion
Using zig types and type reflection allows us to create code which calls dbus look very clean and can be written purely with data of arguments rather than what we would get relying on the raw C interfaces.
This library is still under active development and this post doesn't cover several topics in the code such as parsing dbus arguments into zig and handling registering objects which may be discussed in a future post.
If you want to follow the developement of this library and further explorations in the linux desktop environment or see the code for yourself, head over the [github](https://github.com/sesh9096/wayland-widgets).
I hope this post was of some interest to you.
