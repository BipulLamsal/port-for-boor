+++
title = 'My Summer at VLC - GSoC26'
date = 2026-06-10T11:00:00-07:00
draft = false
tags = ['C','Lua', 'VLC', 'Open Source']
+++

<!--
During my summer, I got the opportunity to work on one of the OG open-source software projects on the internet: VLC. My GSoC project was about building a test runner for Lua modules with a mocked backend (I'll get into what this means later). I would like to thank my mentor, Alexandre Janniaux, for helping me get into the VLC codebase, understand how tests are written, how VLC works internally, its modules and object trees, and for giving me ideas about the project itself, along with some useful tooling such as Gcov and Jujutsu.
-->


# My understanding on VLC plugin based modules
VLC core src (libvlc) is relatively small; everything else (codecs, demuxers, access, lua, etc) exists as seperate shared object (plugins). Even we can create the module, with the macro : `vlc_module_begin()`, which is a compile time macro that lets us declare a module (inside a plugin, the shared object) with its name, a capability (demux, access, etc) with priority score, and callbacks to open and close the module. A single shared object can contain multiple sub module. see for example for [lua module](https://code.videolan.org/videolan/vlc/-/blob/master/modules/lua/vlc.c?ref_type=heads#L653). VLC finds its plugins from plugins directory and opens all the dynamic libraries as needed. On runtime `module_need()` is used to load the module based on the capabilities, based on the score it goes trigger the open callbacks until it finds the desired module. On a high level this is how vlc plugin loads and works together.             

# What's actual lua modules in VLC
Lua module is one of the many modules that exists in the VLC. It registers different submodules with different capabilities.
- lua-extension (cap = `extension`) - runs ui based extensions made from lua script. (vlc ui : tools > plugins) [see examples](https://code.videolan.org/videolan/vlc/-/tree/master/share/lua/extensions?ref_type=heads)
- lua-playlist (cap = `demux`) - probes the access, and parses playlist items based on parser written on the lua script. [see examples](https://code.videolan.org/videolan/vlc/-/tree/master/share/lua/playlist?ref_type=heads)
-  lua-services-discovery (cap = `services_discovery` ) - powers Services Discovery entries in the VLC sidebar. [see examples](https://code.videolan.org/videolan/vlc/-/tree/master/share/lua/sd?ref_type=heads)
- lua-meta - Lua meta modules: art, fetcher and reader that is used to fetch album art or metadata from online sources.

In some way they are playing essentially role manging different parts of VLC using the lua script.

# The problem statement
VLC's playlist module is a demux layer with lower priority than standard demuxers (e.g., mp4). It handles input streams that are playlists (m3u) or HTTP markup, and it runs Lua scripts to parse these markups or HTTP responses into a list of playlist items which the demux pipeline can then consume  [see how playlist is used to load as demux](https://code.videolan.org/videolan/vlc/-/blob/master/test/modules/lua/playlist_parser.c?ref_type=heads#L63).

Lua extensions serve a related but distinct purpose: they let developers build Qt UI dialogs directly in Lua. For example, a dialog might include a search field that scrapes a webpage and returns a list of playlist items to the user.

In both of these scenarios: playlist-parsing scripts and extension scripts the underlying problem is that they depend on external markup that can change at any time, silently breaking the script. Because there is no automated way to verify a script's behavior, we typically only find out it's broken when a user reports it.

This is especially problematic for extensions, since verifying that a dialog's script still parses and returns the correct items currently requires building VLC, launching the Qt interface, and manually clicking through the UI. This workflow cannot be automated in CI, since it depends on live network requests and a running Qt interface.

# What do we need then?
We need a lightweight, standalone test runner that can run a Lua script as the Device Under Test (DUT), independent of a VLC build, with a mocked backend replacing the real network and Qt dependencies.So, this tool should:
 
- Since the DUT is the Lua script itself and not the plugin system, demux layer, or VLC internals. The runner needs to provide the same [Lua modules]((https://code.videolan.org/videolan/vlc/-/tree/master/modules/lua/libs?ref_type=heads)) VLC exposes, but mock out what those calls do at the C layer.
-  To assert expected behavior, each DUT script needs a corresponding test script that drives it and checks its output.
- A tester should be able to run tests in isolation against existing fixtures (e.g., recorded HTTP responses). If a fixture doesn't exist yet, or needs updating, the tester should be able to record a new one directly.

For example: For a playlist parser

```lua
-- DUT : youtube.lua

function probe()
    return vlc.access == "https" and vlc.path == "youtube.com"  
end

function parse()
    -- parses 
    return {--items}
end

```
It should have corresponding test script, that should be able to select the lua script and test it. 

```lua

function test_parsed_item()
    local parser = vlc.test.playlist_parser("youtube.lua")
    assert(parser:probe("https://youtube.com"))
    assert(#parser:parse() == 1)
end

```
Since our test runner should run the DUT it should have access to `vlc.*` so we are not mocking that, we are reusing it. On a high level the diagram looks like this:

![Diagram](./1.png)

# Test Harness for Lua Module

I have one integration test for the playlist parser merged upstream. Its purpose is to verify that the Lua module works correctly. It runs on a libvlc instance, creating a fake interface through which the stream filter module is loaded, and a test script triggers what the module is supposed to do.

For the [stream filter](https://code.videolan.org/videolan/vlc/-/blob/master/modules/lua/stream_filter.c?ref_type=heads), the main job is to trigger `probe()` for the stream, verifying its access, path, or URI. If the probe value is true, meaning our Lua script can accept the stream, we trigger `parse()`, which should return playlist items.

In testing terms:

Setup: create a fake in-memory stream carrying the sample markup, and set its mock URL..
Exercise: this happens in two steps. First, demux_New loads the luaplaylist demuxer, which internally runs probe() on the stream. Then, vlc_stream_ReadDir runs parse() on it
Verification: also happens in two steps, right after each exercise step. First we check that the demuxer was created (probe succeeded), then we check the parsed output: the number of items returned and their values
Cleanup: delete the demux, which internally unloads the module and deletes the stream as well

In this type of integration test, our DUT is the [stream filter](https://code.videolan.org/videolan/vlc/-/blob/master/modules/lua/stream_filter.c?ref_type=heads) module.

By definition, a test harness should let a user test their script with their own test data in a controlled environment. But with the approach above, a Lua script author needs to know the internal C API to properly run and verify their script, and needs to rebuild VLC every time they want to run a test. Even when something breaks, we would not know which part of the Lua script actually caused it. This is why we need a separate driver to execute the Lua script: a test runner.

A test runner's job is to execute the Lua script directly, which makes the DUT more user centric. Our test runner needs the same underlying API a real Lua script relies on. For example, `vlc.stream` should be exposed the same way it is in the case above.

```meson
# pseudocode, simplified for clarity
playlist_parser_source = files('../modules/lua/stream_filter.c')

executable('vlc-lua-mock',
    sources: ['src/main.c', playlist_parser_source],
    include_directories: [
        # includes vlc.h, libs.h from modules/lua
    ],
    c_args: common_args,
    dependencies: [lua_dep], # includes lua dependencies
    )
```

We reuse the playlist parser Lua module so that our Lua script's behavior does not diverge from the real implementation. Linking it this way results in a number of undefined references, which is expected, since the core VLC API is not linked yet for `stream_filter.c`.

For example:

```c
// pseudocode, simplified for clarity
static int vlclua_demux_read( lua_State *L )
{
    stream_t *s = (stream_t *)vlclua_get_this(L); // VLC API, undefined reference
    int n = luaL_checkinteger( L, 1 );
    char *buf = malloc(n);

    if (buf != NULL)
    {
        ssize_t val = vlc_stream_Read(s->s, buf, n); // VLC API, undefined reference
        if (val > 0)
            lua_pushlstring(L, buf, val);
        else
            lua_pushnil( L );
        free(buf);
    }
    else
        lua_pushnil( L );

    return 1;
}
```

The compiler tells us exactly which VLC APIs are touched by the Lua module. We can then define our own versions of functions like `vlc_stream_Read`, backed by our own private struct, close to the actual behavior. Some VLC APIs are unrelated to the VLC object instance, so we can plug those in directly and reuse them as is. That is essentially the gist of the mocked backend.

In testing terms:

- Setup: set up the script (DUT), the fixtures we need, and the test script that will test it
- Exercise: use our test runner to run the script through the same Lua module logic, bypassing the VLC layer, saving and checking state from the mocked environment instead
- Verification: a separate test script verifies the states the script produced, and whether it did what it was supposed to do
- Cleanup: the Lua module cleans up

[For more on the theory behind this, here is what I learned about mocking and stubbing techniques.](https://martinfowler.com/articles/mocksArentStubs.html)

Interesting part here is the testing script, that allows a scipt author to assert and control the VLC environment such as dialogs, playlist,etc. they want via lua script.

# Lua test framework
-- todo
