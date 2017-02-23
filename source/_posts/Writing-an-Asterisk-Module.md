---
title: Writing an Asterisk Module
date: 2014-07-29 20:19:47
tags:
    - Asterisk
    - Tutorial
---

A lot of modules for Asterisk were written a long time ago. For the most part, these modules targeted Asterisk 1.4. I tend to suspect this was for a variety of reasons: the Asterisk project really took off during that time frame, and a lot of capabilities were being added at that time. Unfortunately (or fortunately, if you happen to be one of those who develop for Asterisk often), the Asterisk Architecture has changed a lot since then. While a some of those modules may still work just fine (with maybe a few find and replaces), there's a lot of tools in the toolbox that didn't exist then.

In fact, if I could point to <strong>two</strong> things that have resulted in Asterisk evolving into the stable, robust platform that it is today, it would be:

* **LOTS** of testing.

* Frameworks, frameworks, frameworks!

Not that there is a framework for everything, mind you. Or that everything is "perfect". Software is hard, after all. But if there's something you want to do, there's probably something available to help you do that, and if you use that, then you stand a good chance of using something that just. Plain. Works.

So, let's start at the beginning.

When Asterisk starts, it looks for modules to load in the modules directory (specified in `asterisk.conf` in the `astmoddir` setting). If `modules.conf` says to load a module in that directory - either explicitly or via `autoload` - then it loads that module into memory. But that doesn't actually start the module - it just gets it into memory. However, due to some fancy usage of the `constructor` attribute, it also gets it registered in a few places as a "module" that can be manipulated. That registration really occurs due to an instance of a certain struct that all modules must have, and which provides a few different things:

1. Notification that the module is GPLv2. Unless your module includes the disclaimer that the module is GPLv2, it can't be loaded. So, yes, you have to distribute your module per the conditions of the GPLv2.

2. Various bits of metadata - the name of the module, a description of it, etc.

3. The module's load order priority. Asterisk's module loader is a tad ... simple... which means things have a rough integer priority for loading. If you have lots of dependencies, make sure you load after them - and if things depend on your module, make sure they load after you.

4. Most importantly (for this blog post), it defines a set of virtual functions that will be called at key times by Asterisk. Namely:

    - `load_module`: Called during module load. This should process the configuration, set up containers, and initialize module data. If your module can't be loaded, you can tell the core not to load your module.

    - `unload_module`: The analogue of load_module; called when Asterisk is shutting down. Your module should shut down whatever it is using, unsubscribe from things, dispose of memory, etc.

    - `reload_module`: Called when a module is reloaded by an interface, i.e., through the CLI's module reload command or the AMI ModuleReload action. You should reprocess your configuration when this is called. More on that in a future post; for now, we'll just focus on load and unload.

### Writing the basics

For now, let's just get something that will load. We'll add interesting stuff later. I'm going to assume that this module is named `res_sample_module.c`, stored in the `res` directory of Asterisk. If you'd rather write an application, or a function, or whatever, feel free to rename it and place it where it suits you best. The Asterisk build system should find it, so long as it is in one of the canonical directories. I'm also going to assume that this is done on Asterisk 13 - if you'd like to use another version of Asterisk, that's fine, but your mileage may vary.

1. Add a `MODULEINFO` comment block at the top of your file. When Asterisk compiles, it will parse out this comment block and feed it into menuselect, which allows users to control which modules they want to build. It also will prevent a module from compiling if its dependencies aren't met. For now, we'll simply set its support level:
```
/*** MODULEINFO
     <support_level>extended</support_level>
***/
```

2. Include the main asterisk header file. This pulls in the build options from the configure script, throws in some forward declarations for common items, and gives us the ability to register the version of this file. Immediately after this, we should go ahead and add the call that will register the file version, using the macro `ASTERISK_FILE_VERSION`:
```
#include "asterisk.h"

ASTERISK_FILE_VERSION(__FILE__, "$Revision: $")
```

3. While we're here, after the `ASTERISK_FILE_VERSION`, let's include the header `module.h`. That header contains the definition for the module struct with its handy virtual table of load/unload/reload functions.
```
ASTERISK_FILE_VERSION(__FILE__, "$Revision: $")

#include "asterisk/module.h"
```

4. At the bottom of the file, declare that this is a "standard" Asterisk module. This will make a few assumptions:

    - That you have a `load_module` and `unload_module` static functions conforming to the correct signature, and no `reload_module` function (which is okay, we'll worry about that one another time).

    - That you're okay with loading your module at the "default" time, i.e., `AST_MODPRI_DEFAULT`. For now, we are.

    - That your module generally doesn't get used by other things, e.g., it doesn't export global symbols, or have any really complex dependencies. Again, if we need that, we'll worry about that later.
```
AST_MODULE_INFO_STANDARD(ASTERISK_GPL_KEY, "Sample module");
```

5. Now that we have our sample module defined, we need to provide implementations of the `load_module` and `unload_module` functions. Each of these functions takes in no parameters, and returns an `int`:
```
static int unload_module(void)
{

}

static int load_module(void)
{

}
```

6. These two functions should probably do something. `unload_module` is pretty straight forward; we'll just return 0 for success. Note that generally, a module unload routine should return 0 unless it can't be unloaded for some reason. If the CLI is attempting to unload your module, it will give up; if Asterisk is shutting down, it won't care. It will eventually skip on by your module and terminate (with extreme prejudice, no less).
```
static int unload_module(void)
{
    return 0;
}
```

7. Now we need something for `load_module`. Unlike the unload routine, the load routine can return a number of different things to instruct the core on how to treat the load attempt. In our case, we just want it to load up. For now, we'll just return success:
```
static int load_module(void)
{
    return AST_MODULE_LOAD_SUCCESS;
}
```

### Putting it all together

The whole module would look something like this:

```
/*** MODULEINFO
    <support_level>extended</support_level>
***/

#include "asterisk.h"

ASTERISK_FILE_VERSION(__FILE__, "$Revision: $")

#include "asterisk/module.h"

static int unload_module(void)
{
    return 0;
}

static int load_module(void)
{
    return AST_MODULE_LOAD_SUCCESS;
}

AST_MODULE_INFO_STANDARD(ASTERISK_GPL_KEY, "Sample module");
```

Note that this module is available in a GitHub repo:

https://github.com/matt-jordan/asterisk-modules

### Let's make a module

Let's get this thing compiled and running. Starting from a clean checkout:

1. Configure Asterisk. Note that when you're writing code, always configure Asterisk with `--enable-dev-mode`. This turns gcc warnings into errors, and allows you to enable a number of internal nicities that help you test your code (such as the unit test framework).
```
$ ./configure --enable-dev-mode
```

2. Look at menuselect and make sure your module is there:
```
$ make menuselect
```

    If you don't see it, something went horribly wrong. Make sure your module has the `MODULEINFO` section.

3. Build!
```
$ make
```

4. Install!
```
$ sudo make install
```

5. Start up Asterisk. We should see our module go scrolling on by:
```
$ sudo asterisk -cvvg
...
Loading res_sample_module.so.
== res_sample_module.so => (Sample module)
```

    - We can unload our module:
```
*CLI> module unload res_sample_module.so 
Unloading res_sample_module.so
Unloaded res_sample_module.so
```

    - And we can load it back into memory:
```
*CLI> module load res_sample_module.so
Loaded res_sample_module.so
Loaded res_sample_module.so => (Sample module)
*CLI>
```

Granted, this module doesn't do much, but it does let us form the basis for much more interesting modules in the future.
