---
layout: post
title: Linking Frameworks With DMD
tags:
- programming
- d
- osx
- linking
- framework
---
Little tip for OS X [D Programming Language][1] users: if you want to link with
frameworks, the correct flags for `dmd` are:

    dmd -L-framework -LCoreFoundation test.d

This goes in general for any linker flags with arguments, because `dmd` uses
[-Xlinker][2] to pass the arguments to `ld`.


[1]: http://dlang.org
[2]: http://gcc.gnu.org/onlinedocs/gcc/Link-Options.html