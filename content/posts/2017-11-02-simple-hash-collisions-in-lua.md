---
title: "Simple Hash Collisions in Lua"
description: "It is trivial to generate colliding hashes and create json that takes a non-linear time to process compared to a normal json file."
uuid: "26a15fc6-fb83-4c70-a47d-e4f536a6375a"
date: 2017-11-02
lastmod: 2017-11-02 21:24:40 -0000
slug: "simple-hash-collisions-in-lua"
tags: ["dos", "security"]
categories: ["star", "rss"]
image: "images/posts/simple-hash-collisions-in-lua.png"
image_width: 1006
image_height: 334
---
After investigating the CRuby source code for hash tables, my interest was piqued, and I decided to look into another hash table implementation before leaving Recurse Center. I chose to investigate Lua because the hash table is the central abstraction of the language. I don't actually know Lua, but luckily I spent most of my investigation in a debugger or C source code.

One of the first things that jumped out at me was that there didn't seem to be any mention of `siphash` in the Lua source code. Ruby and Python use `siphash` with a per-session key to generate hashes. You can easily verify that the hash of a string changes in different Ruby sessions:

{{< highlight bash >}}
$ irb
irb(main):001:0> "hello!".hash
=> -3270480275822396260
irb(main):002:0> exit
$ irb
irb(main):001:0> "hello!".hash
=> -4583553666733776547
{{< /highlight >}}

Using a debugger, I saw that strings in Lua also have different hashes between different sessions. However, Lua doesn't appear to use `siphash`, so I decided to investigate exactly what it does instead. The Lua hashing approach for strings is nearly at the top of `lstring.c`:

{{< highlight c >}}
unsigned int luaS_hash (const char *str, size_t l, unsigned int seed) {
  unsigned int h = seed ^ cast(unsigned int, l);
  size_t step = (l >> LUAI_HASHLIMIT) + 1;
  for (; l >= step; l -= step)
    h ^= ((h<<5) + (h>>2) + cast_byte(str[l - 1]));
  return h;
}
{{< /highlight >}}

Reading this code, it is pretty obvious that most of the string is ignored when it is hashed. This makes it very easy to generate collisions! For example, in Lua, all these strings have the same hash:

{{< highlight sh >}}
"0000000000000000000000000000000000"
"f0l0l0w0m0e0n0t0w0i0t0t0e0r0?0:0)0"
"x0x0x0x0x0x0x0x0x0x0x0x0x0x0x0x0x0"
{{< /highlight >}}

I wrote up a quick script to generate collisions and tested with 50,000 values of length 34 compared to random strings in a similar setup. Parsing a json file that uses these strings as keys with rapidjson (a popular Lua library) showed stark results:

{{< highlight sh >}}
$ lua parse-json.lua
0.04s user 0.01s system 82% cpu 0.067 total
$ time lua parse-json-collision.lua
13.68s user 0.07s system 99% cpu 13.841 total
{{< /highlight >}}

The test with colliding hashes took **more than 300&times; longer** to process. The growth is not linear: a larger file could take significantly longer.

This problem of Hash DoSing is mentioned on [lua-users Wiki](http://lua-users.org/wiki/HashDos), and there is discussion about this on their mailing in 2012. However, in 2017 using the latest version of Lua, it is still trivial to generate collisions.

Thank you to [Wesley](http://blog.wesleyac.com/) and Jake for pairing on debugging Lua, and reading the source code.
