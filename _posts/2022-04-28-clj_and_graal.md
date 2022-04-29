---
title: Clj + Graal: Compiling Clojure a static binary.
author: Scott
layout: post
tags: meta
---

I have been plugging away at [a first implementation of Ludus](https://github.com/thinking-with-computers/cludus) for a while now, and it's both quite nice in terms of syntax, and it's Turing-complete! (All of the gory details of my design process and implementation state are available in the [ludus-spec repo](https://github.com/thinking-with-computers/ludus-spec) on Github.) I have much more to say about Ludus and its design in the (near) future (just once this term ends and I revise my second book...). 

Today, I built a native binary for Ludus for the first time. This post is just documenting (for my future self, for my collaborators, for the internet at large) what I did to get it working, since it's a bit titchy. (tl;dr: Read all of the "Caveats" `brew` throws your way; they have crucial information. This post would not be nearly so necessary if I had RTFM.)

First, as a disclaimer, I should say that my setup is a bit specific, and not all steps are quite this annoying if you're not in my exact situation. I'm running MacOS Monterey (12.3) on an M1 MacBook Pro, and using fish (instead of zsh or bash) as my shell. 

Also, for early versions of Ludus, we're using Clojure as the implementation language. That's mostly because (1) it's a Lisp, and we like Lisps here at Thinking with Computers, and (2) Ludus's persistent, immutable data structures are modeled off of Clojure's; it's semantically the most similar language to Ludus. Clojure's great, but it runs on the JVM by default and therefore requires some JVM acrobatics that are unfamiliar to me (who has mostly lived in the JS ecosystem).

### What is GraalVM?
[GraalVM](https://www.graalvm.org/) is a polyglot virtual machine that, among other things, allows for building (fast) static binaries for interpreted languages. It's built by Oracle (ugh). For our purposes, it takes JVM programs and turns them into binaries.

### Step 0: Prerequisites
I assume you have [Homebrew](https://brew.sh) installed and are handy enough with the command line.

I also assume you have Clojure and Leiningen installed. If you don't:

`> brew install clojure leiningen`

...and voilà!

### Step 1: Get jenv
Jenv is a Java version/environment manager. You will be installing a new Java runtime (GraalVM is also a runtime). Rather than dealing with manually working with Java environments, we use jenv: 

`brew install jenv`.

Jenv will need to have various configs installed, so heed the Caveats. Running Fish, I see I need to complete setup by running:

`> echo 'status --is-interactive; and source (jenv init -|psub)' >> ~/.config/fish/config.fish`.

Copy-pasta!

To reload Fish with our new path: `source ~/.config/fish/config.fish`.

### Step 2: Get GraalVM & `native-image`
GraalVM is hidden behind a cask: 

`> brew install --cask graalvm/tap/graalvm-ce-java17`.

If you're on a recent version of MacOS, the whole install of Graal violates Apple's attempts to keep you from running unsafe code/malware/bleeding edge tech/hobbyist programs. That means you'll have to disable SIP for this particular install: 

`> xattr -r -d com.apple.quarantine "/Library/Java/JavaVirtualMachines/graalvm-ce-java17-22.1.0"`

Once Graal is downloaded and installed, we need to set it up by adding it to jenv:

`> jenv add /Library/Java/JavaVirtualMachines/graalvm-ce-java17-22.1.0/Contents/Home/`

(If MacOS is complaining that the software is corrupt and should be trashed, make sure you did the `xattr` command above properly.)

Finally, we have to tell jenv that we want to use Graal. Because Graal for the M1 is still experimental, I figure it's a bad idea to use it as your default JVM. So, for our purposes: 

`> jenv shell graalvm64-17.0.3`

(If you wanted to use it as your default, you'd replace `shell` with `global`. If you use `shell`, you'll have to run this every time you want to use Graal to compile a static binary. Thankfully, jenv includes fish tab-completion, so you can `jenv shell graal<tab>` and you're good to go.)

To make sure you've done everything right, run `java --version`, and you'll see something like:

```
openjdk 17.0.3 2022-04-19
OpenJDK Runtime Environment GraalVM CE 22.1.0 (build 17.0.3+7-jvmci-22.1-b06)
OpenJDK 64-Bit Server VM GraalVM CE 22.1.0 (build 17.0.3+7-jvmci-22.1-b06, mixed mode, sharing)
```

One last sub-step! Graal does not come with its `native-image` utility installed. It's what we'll need to make a native binary. We use `gu`--Graal update--to install it.

`> gu install native-image`

Bingo! We have a compiled-Java-to-native-binary compiler. Just to test, `native-image --version`.

### Step 3: Compile some Clojure
By default, Leiningen projects come close to working with `native-image`, but not quite. In addition, the incantation to `native-image` is a doozie. So, let's get started.

#### Step 3a: `:gen-class`
In your `core.clj` file, with the `-main` entry point, you need to ensure that the `ns` form includes `:gen-class`, which will instruct Clojure to create a Java class file for your program, e.g.:

```clojure
(ns ludus.core
	; requires omitted
	(:gen class))
```

#### Step 3b: Edit your `project.clj`
Assuming you've used Leiningen for project management, your `project.clj` filly will look something like:

```clojure
(defproject ludus "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "EPL-2.0 OR GPL-2.0-or-later WITH Classpath-exception-2.0"
            :url "https://www.eclipse.org/legal/epl-2.0/"}
  :dependencies [[org.clojure/clojure "1.10.3"]]
  :repl-options {:init-ns ludus.core}
 )
```

You'll need to add two things to this: you'll need to specify the entry point to the program (in the form of a `-main` function), and you'll need to tell Leiningen to do all the ahead-of-time (AOT) compilation. You will add two lines to the `defproject` form (after `:repl-options`, before the close paren):

```clojure
:main ludus.core ; specifies the file with `-main`
:profiles {:uberjar {:aot :all}} ; aot all the things
```

#### Step 3c: Compile your program to a `.jar`
Once you've done that, tell Leingingen to compile your project:

`> lein uberjar`

This will create two files in your `target/` directory (assuming the project name is `ludus`): `ludus-0.1.0-SNAPSHOT.jar` and `ludus-0.1.0-SNAPSHOT-standalone.jar`. It's the `standalone` file we're interested in.

#### Step 3d: Create a native image
Now we have what we need for a native image:

```
> native-image --report-unsupported-elements-at-runtime \
                      --initialize-at-build-time \
                      -jar ./target/ludus-0.1.0-SNAPSHOT-standalone.jar \
                      -H:Name=./target/ludus
```

Obviously, change this for your `.jar` filename, version number, and binary name. And voila! You have an executable binary.

One note: the `--initialize-at-build-time` flag is deprecated, and supposed to have been removed in GraalVM v22. I'm using v22.1, and it's still working (but with a deprecation warning saying it'll be removed in v22.) It's necessary for Clojure projects. There is a great deal of discussion (much of it over my head; I'm not a JVM person) in [the GraalVM issues](https://github.com/oracle/graal/discussions/3476). The good folks at clj-easy apparently [have a workaround](https://github.com/clj-easy/graal-build-time), but I haven't tried it yet.

### Conclusions
The static binary is fast![0] It works. It's frankly magic for me to have written a programming language, and then to have a static binary that runs that language. We'll get to other kinds of magic later! Frankly, the magic has yet to stop.

[0] Erm, for some value of "fast." It's way faster than `lein run` in any event. The Clojure implementation of Ludus is very slow. It's a tree-walk interpreter that has no optimization of any kind at this point. But such interpreters are faster to develop and easier to change than the eventual bytecode interpreter written in a lower-level language.

