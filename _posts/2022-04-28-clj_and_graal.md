---
title: Clojure + GraalVM.
author: Scott
layout: post
tags: meta
---

I have been plugging away at a first implementation of Ludus for a while now, and it's both quite nice in terms of  and Turing-complete. (All of the gory details of my design process and implementation state are available in the [ludus-spec repo](https://github.com/thinking-with-computers/ludus-spec) on Github.) I have much more to say about Ludus and its design in the future. 

Today, I built a native binary for Ludus for the first time. This post is just documenting (for my future self, for my collaborators, for the internet at large) what I did to get it working, since it's a bit titchy. (tl;dr: Read all of the "Caveats" `brew` throws your way; they have crucial information. This post would not be necessary if I had RTFM.)

First, as a disclaimer, I should say that my setup is a bit specific, and not all steps are quite this annoying if you're not in my exact situation. I'm running on an M1 MacBook pro, and using fish (instead of zsh) as my shell. Also, for early versions of Ludus, we're using Clojure as the implementation language. That's mostly because (1) it's a Lisp, and we like Lisps here at Thinking with Computers, and (2) Ludus's persistent, immutable data structures are modeled off of Clojure's; it's semantically the most similar language to Ludus.

### What is GraalVM?
[GraalVM](https://www.graalvm.org/) is a polyglot virtual machine that, among other things, allows for (fast) native binaries for interpreted languages. It's built by Oracle (ugh). For our purposes, it takes JVM programs and turns them into binaries.

### Step 0: Prerequisites
I assume you have [Homebrew](https://brew.sh) installed and are handy enough with the command line.

I also assume you have Clojure and Leiningen installed. If you don't:

`brew install clojure leiningen`

### Step 1: Get jenv
Jenv is a Java version/environment manager. You will be installing a new Java runtime (GraalVM is also a runtime). Rather than dealing with manually working with Java environments, we use jenv: 

`brew install jenv`.

Jenv will need to have various configs installed, so heed the Caveats. Running Fish, I see I need to complete setup by running:

`echo 'status --is-interactive; and source (jenv init -|psub)' >> ~/.config/fish/config.fish`.

Copy-pasta!

To reload Fish with our new path: `source ~/.config/fish/config.fish`.

### Step 2: Get GraalVM & `native-image`
GraalVM is hidden behind a cask: 

`brew install --cask graalvm/tap/graalvm-ce-java17`.

If you're on a recent version of MacOS, the whole install of Graal violates Apple's attempts to keep you from running unsafe code/malware/bleeding edge tech/hobbyist programs. That means you'll have to disable SIP for this particular install: 

`xattr -r -d com.apple.quarantine "/Library/Java/JavaVirtualMachines/graalvm-ce-java17-22.1.0"`

Once Graal is downloaded and installed, we need to set it up by adding it to jenv:

`jenv add /Library/Java/JavaVirtualMachines/graalvm-ce-java17-22.1.0/Contents/Home/`

(If MacOS is complaining that the software is corrupt and should be trashed, make sure you did the `xattr` command above properly.)

Finally, we have to tell jenv that we want to use Graal. Because Graal for the M1 is still experimental, I figure it's a bad idea to use it as your default JVM. So, for our purposes: 

`jenv shell graalvm64-17.0.3`

(If you wanted to use it as your default, you'd replace `shell` with `global`.)

To make sure you've done everything right, run `java --version`, and you'll see something like:

```
openjdk 17.0.3 2022-04-19
OpenJDK Runtime Environment GraalVM CE 22.1.0 (build 17.0.3+7-jvmci-22.1-b06)
OpenJDK 64-Bit Server VM GraalVM CE 22.1.0 (build 17.0.3+7-jvmci-22.1-b06, mixed mode, sharing)
```

One last sub-step! Graal does not come with its `native-image` utility installed. It's what we'll need to make a native binary. We use `gu`--Graal update--to install it.

`gu install native-image`

Bingo! We have a compiled-Java-to-native-binary compiler. Just to test, `native-image --version`.

### Step 3: Compile some Clojure
By default, Leiningen projects come close to working with `native-image`, but not quite. In addition, the incantation to `native-image` is a doozie. So, let's get started.

#### Step 3a: `:gen-class`
In your `core.clj` file, with the `-main` entry point 