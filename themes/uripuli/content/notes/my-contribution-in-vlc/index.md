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
VLC's playlist module is a demux layer with lower priority than standard demuxers (e.g., mp4). It handles input streams that are playlists (m3u) or HTTP markup, and it runs Lua scripts to parse these markups or HTTP responses into a list of playlist items which the demux pipeline can then consume  [see how playlist is used to load as demux] (https://code.videolan.org/videolan/vlc/-/blob/master/test/modules/lua/playlist_parser.c?ref_type=heads#L63).

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


-- todo
