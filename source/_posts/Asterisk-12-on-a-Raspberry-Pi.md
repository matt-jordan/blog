---
title: Asterisk 12 on a Raspberry Pi
date: 2013-09-02 07:06:26
tags:
    - Asterisk
---

So, like a lot of people, as soon as I could I got my hands on a Raspberry Pi (I got mine from [Adafruit](https://www.adafruit.com/) and was very happy with the experience). If you don't know what a Raspberry Pi is, it's the greatest thing since sliced bread - a small single board computer that runs Linux for only thirty-five bucks. Along with a case, a good power supply, and a few other odds and ends, it's still a steal, coming in somewhere under seventy-five bucks. They're ridiculously flexible. They can be used for a whole host of microcontroller projects, but are also powerful enough to act as mini home "servers". For awhile now, I've planned to set mine up as a home Asterisk system. In particular, my poor wife - who happens to work from home - has had to make do without much in the way of good business communications (there's a story in there somewhere about a cobbler and his kids not having any shoes). Unfortunately, just about every waking moment for the past six months has been spent on getting Asterisk 12 developed and released, so my poor Pi has sat on a desk collecting dust.

Today, however, no more. I got the Pi out, plugged it in, connected it to my home network, and got tinkering. The goal: get Asterisk 12 running on a Raspberry Pi. By running, I mean just running - configuration is a bit beyond the scope of today (or this blog post) - but if I can get myself to the CLI prompt, that will do. As an aside, this is realistic: the Raspberry Pi does not compile quickly. I ended up running a lot of errands between steps, so this whole project was stretched out quite a bit over today.

It's not often that a developer gets to take a step back and look at things from a user's perspective, so this should be a lot of fun.

As another aside, I think like most folks using a Raspberry Pi for the first time, I found the experience to be quite a pleasure. While I would never claim that I'm a Linux guru (I did work in Microsoft shops for quite a long time; I still find myself moving back and forth between the two worlds), I can get around just fine in the major Linux distros and have no problems setting up a Linux system. Even still, I was still pleasantly surprised at the configuration tools that come with the Pi. They really nailed their target audience, and even for those of us who use Linux daily, they still made life nice and easy.

Moving on!

I spent the first bit just configuring the Pi with the tools. I set it up with a static IP address behind my home router, updated Wheezy, changed the default user's password to something secure, and changed the hostname to 'mjordan-pi' (terribly original, I know. I'm not good at naming things - I leave that to Mr. Joshua Colp on our team)

At this point, I figured I should be good to go on the actual task of getting Asterisk downloaded, installed, and configured. While the folks at Asterisk for Raspberry Pi have done a spectacular job of documenting and making it easy to install Asterisk on a Pi, I'm going to depart from their guidelines here. While I love FreePBX, I'm going to eschew a GUI as well as MySQL. I really want a relatively stream-lined install of Asterisk 12, with .conf files for configuration, access through the CLI, and everything done by hand. When we get to configuration, you'll note that I'm going to take a pretty slow and conservative approach to configuration - but that will let me look at each module and step and really digest what I'm doing. We're going old school here. Should be fun!

# Step 1: Get Asterisk

Since I'm working through ssh from my laptop, everything is going to be done through the shell. That means downloading Asterisk using wget from downloads.asterisk.org.

```
pi@mjordan-pi ~ $ wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-12-current.tar.gz
--2013-09-01 16:27:28-- http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-12-current.tar.gz
 Resolving downloads.asterisk.org (downloads.asterisk.org)... 76.164.171.238, 2001:470:e0d4::ee
 Connecting to downloads.asterisk.org (downloads.asterisk.org)|76.164.171.238|:80... connected.
 HTTP request sent, awaiting response... 200 OK
 Length: 56720157 (54M) [application/x-gzip]
  Saving to: `asterisk-12-current.tar.gz'
  100%[====================================================================================================>] 56,720,157 982K/s in 61s
  2013-09-01 16:28:29 (905 KB/s) - `asterisk-12-current.tar.gz' saved [56720157/56720157]
```

Hooray! We got it. Now to untar it:

```
pi@mjordan-pi ~ $ tar -zvxf asterisk-12-current.tar.gz
```

As you may notice, Asterisk 12 is a bit large (what can I say, we did a lot of work). Some of that heft comes from the exported documentation from the Asterisk wiki, in the form of the Asterisk Administrators Guide. Since I don't really need that on my Pi, I'm going to go ahead and get rid of it, as well as the tarball now that I've extracted it.

```
 pi@mjordan-pi ~ $ rm asterisk-12-current.tar.gz
 pi@mjordan-pi ~ $ rm asterisk-12.0.0-alpha1/Asterisk-Admin-Guide-12.html.zip
 pi@mjordan-pi ~ $ rm asterisk-12.0.0-alpha1/doc/Asterisk-Admin-Guide-12.pdf
```

Random note here. You might be wondering why the PDF is in the doc subfolder, while the zip of the HTML documentation is in the base directory. When we were making the Alpha tarball, we had a number of problems crop up in the scripts that make the release. There's a lot of moving parts in that process, in particular with making the release summaries and pulling the documentation from Confluence. We had some connectivity issues going back from the release build virtual machine to the issue tracker, such that the connection would drop - or have some random fit at least - while it was trying to obtain information about Asterisk issues. After the fourth or fifth exception thrown by the script (various socket errors, HTTP timeouts, and other random badness), we had managed to pull all of the data but - for various reasons - not in the correct locations in the directory that was destined to become the tarball. So the location of the zip file is a goof on my part - I had to move it manually, and accidentally moved it into the wrong location. All of this went to show that it's a well known fact that whatever can go wrong in your systems will go wrong just as you're trying to push something live, particularly if you're supposed to meet with your colleagues for a celebratory beer in five minutes. Every. Freaking. Time.

Anyway, let's see how far we get running configure:

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ ./configure
configure: error: *** termcap support not found (on modern systems, this typically means the ncurses development package is missing)
```

That's not terribly surprising. In fact, [raspberry-asterisk.org](http://www.raspberry-asterisk.org/faq/) has a good list of dependencies you'll need if your'e installing Asterisk from source. As it is, I don't really want all of the libraries listed in the FAQ. For example, I know I won't be integrating Asterisk with MySQL, as I am - for the time - eschewing any Realtime configuration. But most of the rest are needed, and, as the Raspberry Pi is slow, it's good to think things through and get things right the first time.

# Step 2: Install Asterisk Dependencies

Let's get what we do know:

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ apt-get install build-essential libsqlite3-dev libxml2-dev libncurses5-dev libncursesw5-dev libiksemel-dev libssl-dev libeditline-dev libedit-dev curl libcurl4-gnutls-dev
```

Since this is Asterisk 12, however, I'll want a few other things as well. The [CHANGES](http://git.asterisk.org/gitweb/?p=asterisk/asterisk.git;a=blob;f=CHANGES;h=2682d85b81c698b39590c82dded1fc06780dfa4b;hb=b0b1b67d3efa6ce4367e39127c56e2fbdb055334) and [UPGRADE.txt](http://git.asterisk.org/gitweb/?p=asterisk/asterisk.git;a=blob;f=UPGRADE.txt;h=2712a325a51f85034ca0e82b2cb1e5e2c0a1521f;hb=b0b1b67d3efa6ce4367e39127c56e2fbdb055334) file tell you the additional dependencies - let's got those as well:

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ apt-get install libjansson4 libjansson-dev libuuid1 uuid-dev libxslt1-dev liburiparser-dev liburiparser1
```

We could, at this point, configure and build Asterisk - but there's one more thing I want. Since this is Asterisk 12, we have the brand new SIP stack to try out, and I'm definitely going to be using it over the legacy chan_sip channel driver. But, to get it working, we'll need PJSIP.

# Step 3: Get PJSIP

As of today (and I'm hopeful this isn't a permanent situation), Asterisk needs a particular flavor of pjproject (I'm going to use the terms pjproject/PJSIP interchangeably here. There is a difference, but let's just pretend there isn't for the purposes of this blog post). There's instructions on the wiki for how to obtain, configure, and install pjproject for use with Asterisk:

https://wiki.asterisk.org/wiki/display/AST/Installing+pjproject

Following along with the instructions, we first need git. Let's get that:

```
 pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ apt-get install git
```

And then we can clone pjproject from our repo on github:

```
 pi@mjordan-pi $ git clone https://github.com/asterisk/pjproject pjproject
 pi@mjordan-pi $ cd pjproject
```

Now here's the tricky part. What parameters should we pass to the configure script?

At a minimum, from the wiki article we know we need to tell it to install in /usr, and to enable shared objects with --enable-shared. But pjproject embeds a lot of third party libraries, which will conflict if we want to use them in Asterisk (or at least, will generally not play nicely). It is very important to know what you have on your system what installing pjproject, and what you want to use in Asterisk.

In my case, I don't have libsrtp installed and I'm not going to use SRTP in Asterisk, so I can ignore that. I also don't have libspeex installed, and I don't really care about using libspeex in Asterisk either. The same goes for the GSM library. So, in the end I ended up with the following:

```
pi@mjordan-pi ~/pjproject $ ./configure --prefix=/usr --enable-shared --disable-sound --disable-video --disable-resample
```

After a little bit, we get the configuration success message:

```
Configurations for current target have been written to 'build.mak', and 'os-auto.mak' in various build directories, and pjlib/include/pj/compat/os_auto.h.
Further customizations can be put in:
 - 'user.mak'
 - 'pjlib/include/pj/config_site.h'
The next step now is to run 'make dep' and 'make'.
```

I went ahead and skipped 'make dep', since we don't really need to do that step. Thus... the fateful command was typed:

```
pi@mjordan-pi ~/pjproject $ make
```

This took a long time.

```
if test ! -d ../bin/samples/armv6l-unknown-linux-gnu; then mkdir -p ../bin/samples/armv6l-unknown-linux-gnu; fi
gcc -o ../bin/samples/armv6l-unknown-linux-gnu/vid_streamutil \
 output/sample-armv6l-unknown-linux-gnu/vid_streamutil.o -L/home/pi/pjproject/pjlib/lib -L/home/pi/pjproject/pjlib-util/lib -L/home/pi/pjproject/pjnath/lib -L/home/pi/pjproject/pjmedia/lib -L/home/pi/pjproject/pjsip/lib -L/home/pi/pjproject/third_party/lib -lpjsua -lpjsip-ua -lpjsip-simple -lpjsip -lpjmedia-codec -lpjmedia -lpjmedia-videodev -lpjmedia-audiodev -lpjnath -lpjlib-util -lmilenage -lsrtp -lgsmcodec -lspeex -lilbccodec -lg7221codec -lpj -lm -luuid -lm -lnsl -lrt -lpthread -lcrypto -lssl
make[2]: Leaving directory `/home/pi/pjproject/pjsip-apps/build'
make[1]: Leaving directory `/home/pi/pjproject/pjsip-apps/build'
```

Huzzah! Let's get this bad boy installed.

```
pi@mjordan-pi ~/pjproject $ sudo make install
```

And after a bit:

```
for d in pjlib pjlib-util pjnath pjmedia pjsip; do \
cp -RLf $d/include/* /usr/include/; \
 done
  mkdir -p /usr/lib/pkgconfig
   sed -e "s!@PREFIX@!/usr!" libpjproject.pc.in | \
    sed -e "s!@INCLUDEDIR@!/usr/include!" | \
     sed -e "s!@LIBDIR@!/usr/lib!" | \
      sed -e "s/@PJ_VERSION@/2.1.0/" | \
`                sed -e "s!@PJ_LDLIBS@!-lpjsua -lpjsip-ua -lpjsip-simple -lpjsip -lpjmedia-codec -lpjmedia -lpjmedia-videodev -lpjmedia-audiodev -lpjnath -lpjlib-util -lmilenage -lsrtp -lgsmcodec -lspeex -lilbccodec -lg7221codec -lpj -lm -luuid -lm -lnsl -lrt -lpthread -lcrypto -lssl!" | \
        sed -e "s!@PJ_INSTALL_CFLAGS@!-I/usr/include -DPJ_AUTOCONF=1 -O2 -DPJ_IS_BIG_ENDIAN=0 -DPJ_IS_LITTLE_ENDIAN=1 -fPIC!" > //usr/lib/pkgconfig/libpjproject.pc
```

Let's verify we got it.

```
pi@mjordan-pi ~/pjproject $ pkg-config --list-all | grep pjproject
libpjproject libpjproject - Multimedia communication library
pi@mjordan-pi ~/pjproject $
```

Yay! As an aside, Asterisk uses pkg-config to locate libpjproject - so if this step fails, something has gone horribly wrong and Asterisk is not going to find libpjproject either. So this is always a useful step to perform, regardless of the platform you're installing Asterisk on.

# Step 4: Build and Install Asterisk

Back to Asterisk!

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ ./configure --with-pjproject
```

I specified `--with-pjproject`, because I really wanted to know if it failed. It takes a bit to compile, and this way the configure script will tell me. Otherwise, the first time I'll find out whether or not I have the dependencies for the new SIP stack is when I fire up menuselect.

```
configure: Menuselect build configuration successfully completed
$$$$$$$$$$$$$$$$.
configure: Package configured for:
configure: OS type : linux-gnueabihf
configure: Host CPU : armv6l
configure: build-cpu:vendor:os: armv6l : unknown : linux-gnueabihf :
configure: host-cpu:vendor:os: armv6l : unknown : linux-gnueabihf :
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $
```

And now for some configuration via menuselect.

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ make menuselect
```

I'm going to be a bit conservative here. There's a lot of things I just won't need for this Asterisk install. Rather than build everything and exclude them all via modules.conf, I'm going to disable them here. That way my overall installation will just be smaller (space is a premium on a pi), and I'll be less likely to leave something lying around in Asterisk that doesn't need to be there.

The following are what I disabled in menuselect:

* Compiler flags:
    - `DONT_OPTIMIZE` - this may seem odd, but since I'm running an alpha of Asterisk 12, if I have a problem, I want to be able to fix it. This will make backtraces a bit more usable.
* Applications:
    - `app_agent_pool` - I'm not going to be using Queues or Agents.
    - `app_authenticate` - I'm not going to need to Authenticate people myself.
    - `app_gelgenuserevent` - I'm not using CEL.
    - `app_forkcdr` - I hate this application, but that's probably because I had to rewrite the CDR engine in Asterisk 12. While there's one or two valid reasons to fork a CDR, most of what ForkCDR has traditionally done is silly.
    - `app_macro` - Use GoSubs everyone!
    - `app_milliwatt` - I'll use something else to annoy people.
    - `app_page` - I don't see myself setting up a home paging system right now. I'm only planning on connecting one or two SIP phones.
    - `app_privacy` - Nope!
    - `app_queue` - If I need a tech support queue for my house, I've got bigger problems.
    - `app_sendtext` - Nope.
    - `app_speech_utils` - Maybe someday, but probably not. I'd rather play around with calendaring or conferencing than speech manipulation of any kind.
    - `app_system` - Until I have a use for it, I view this as one big security vulnerability.
    - Everything in extended, except for `app_zapateller`. It may be fun to try and zap a telemarketer.
* CDR:
    - Everything but `cdr_custom`. I'll probably end up logging call records using this, since we don't tend to have more than a handful of calls a day.
* CEL:
    - Disable everything.
* Channel Drivers:
    - `chan_iax2` - I'm not going to set up an IAX2 trunk, and all my phones are SIP.
    - `chan_multicast_rtp` - Since I'm not doing any paging, I don't need this channel driver.
    - `chan_sip` - I'm committing to `chan_pjsip`!
    - I left `chan_motif` to play around with some day. This may be useful for some soft phones or XMPP text message integration.
    - Everything in the extended/deprecated sections was disabled.
* Codecs:
    - I left all of the codecs enabled.
* Formats:
    - `format_jpeg` - I'm not going to do any fax at this point.
    - `format_vox` - I shouldn't need this either.
* Functions:
  I decided to leave most functions. Most function modules are rather small and don't take up much space, and I find that you never know when you're going to need one.
    - `func_frame_trace` - This is only ever used for development (and isn't used often then)
    - `func_pitchshift` - Funny, but not super useful.
    - `func_sysinfo` - Interesting, but I shouldn't need it.
    - `func_env` - I shouldn't need to muck around with environment variables from the dialplan.
* PBX:
    - `pbx_ael` - Classic dialplan all they way for me.
    - `pbx_dundi` - I don't think I'll be needing anything as complex as DUNDi for my little Pi.
    - `pbx_realtime` - Definitely not. I'm not a fan of realtime dialplan, for a variety of reasons.
* Resources:
  I'm going to leave a number of resources that I may end up using someday, such as res_calendar. The ones I know for sure I won't use I'll disable.
    - `res_adsi` - Nope. I'm not sure you can even find ADSI devices still.
    - `res_config_curl` - Nope, not doing Realtime. I'll come back and enable it if I decide to do Realtime some day.
    - `res_config_sqlite3` - Nope, for the same reason as res_config_curl.
    - `res_fax` - Ew, fax. Not going to mess with fax at home.
    - `res_pjsip_info_dtmf` - I shouldn't need DTMF over INFO.
    - `res_pjsip_endpoint_identifier_anonymous` - I don't want anonymous access.
    - `res_pjsip_one_touch_record_info` - The SIP devices I'm using won't need this.
    - `res_pjsip_t38` - Fax: just say no.
    - `res_rtp_multicast` - Since I'm not doing any paging, I don't need `res_rtp_multicast`.
    - `res_smdi` - Same reason as res_adsi.
    - `res_speech` - Nope, for the same reason as the speech utilities.
    - Everything in extended, except: `res_http_websocket`. That should be fun to play with at some point, and ARI needs it.
* MoH/Sounds:
    - I enabled g.722 sounds/MoH. Wideband audio!

Phew. Time to save changes and finally compile.

```
menuselect changes saved!
make[1]: Leaving directory `/home/pi/asterisk-12.0.0-alpha1'
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ make
...
Building Documentation For: channels pbx apps codecs formats cdr cel bridges funcs tests main res addons
+--------- Asterisk Build Complete ---------+
+ Asterisk has successfully been built, and +
+ can be installed by running: +
+ +
+ make install +
+-------------------------------------------+
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $
```

Woot! We're compiled. Time to install:

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ sudo make install

+---- Asterisk Installation Complete -------+
+ +
+ YOU MUST READ THE SECURITY DOCUMENT +
+ +
+ Asterisk has successfully been installed. +
+ If you would like to install the sample +
+ configuration files (overwriting any +
+ existing config files), run: +
+ +
+ make samples +
+ +
+----------------- or ---------------------+
+ +
+ You can go ahead and install the asterisk +
+ program documentation now or later run: +
+ +
+ make progdocs +
+ +
+ **Note** This requires that you have +
+ doxygen installed on your local system +
+-------------------------------------------+
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $
```

And we're installed!

I'm going to set up the bare minimum Asterisk configuration files to get Asterisk up and running. I'm not going to do anything more than get the CLI prompt up for now - I'll save getting a phone configured and howler monkeys playing back for another time.

# Step 5: Configure Asterisk

## `asterisk.conf`

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ sudo nano /etc/asterisk/asterisk.conf

[options]
verbose = 5
debug = 0
systemname = mjordan-pi
```

As you can see, a pretty simple `asterisk.conf`. Not much else is needed for this, at least for now.

## `modules.conf`

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ sudo nano /etc/asterisk/modules.conf

[modules]
autoload = no

; -- Bridges --

load => bridge_builtin_features.so
load => bridge_builtin_interval_features.so
load => bridge_holding.so
load => bridge_native_rtp.so
load => bridge_simple.so
load => bridge_softmix.so

; -- Formats --

load => format_g719.so
load => format_g723.so
load => format_g726.so
load => format_g729.so
load => format_gsm.so
load => format_h263.so
load => format_h264.so
load => format_ilbc.so
load => format_pcm.so
load => format_siren14.so
load => format_siren7.so
load => format_sln.so
load => format_wav_gsm.so
load => format_wav.so

; -- Codecs --

load => codec_adpcm.so
load => codec_alaw.so
load => codec_a_mu.so
load => codec_g722.so
load => codec_g726.so
load => codec_gsm.so
load => codec_ilbc.so
load => codec_lpc10.so
load => codec_resample.so
load => codec_ulaw.so

; -- PBX --

load => pbx_config.so
load => pbx_loopback.so
load => pbx_spool.so
```

Rather than go with autoloading, I've chosen to explicitly load modules. For now, I've only specified the "basics" - codecs and formats, the bridge modules (now used a lot in Asterisk 12), and the ancillary PBX modules. This will let me catch configuration errors easier - if you autoload, I find most people tend to ignore the warnings and errors.

## `extensions.conf`

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ sudo nano /etc/asterisk/extensions.conf

[general]

[default]
```

Nothing needed in there right now!

# Step 6: Profit

And now, for the moment of truth:

```
pi@mjordan-pi ~/asterisk-12.0.0-alpha1 $ sudo asterisk -cvvvvg

...

Loading codec_adpcm.so.
== Registered translator 'adpcmtolin' from format adpcm to slin, table cost, 900000, computational cost 1
== Registered translator 'lintoadpcm' from format slin to adpcm, table cost, 600000, computational cost 1
codec_adpcm.so => (Adaptive Differential PCM Coder/Decoder)
Loading codec_g726.so.
== Registered translator 'g726tolin' from format g726 to slin, table cost, 900000, computational cost 60000
== Registered translator 'lintog726' from format slin to g726, table cost, 600000, computational cost 120000
== Registered translator 'g726aal2tolin' from format g726aal2 to slin, table cost, 900000, computational cost 60000
== Registered translator 'lintog726aal2' from format slin to g726aal2, table cost, 600000, computational cost 120000
codec_g726.so => (ITU G.726-32kbps G726 Transcoder)
Asterisk Ready.
*CLI>
```

Woohoo!
