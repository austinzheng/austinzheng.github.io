---
layout: page
title: Projects
permalink: /projects/
---

A selected list of projects I've worked on follows.

## Lambdatron ##

**Lambdatron** is a Clojure-like Lisp interpreter implemented in Swift. It is provided as an OS X command-line application that be run either as a REPL or used to execute code files. The eventual goal is to package it as a library which can be dropped into other applications and used as an embedded scripting language, or used with the REPL as a stand-alone interpreter.

While still a work in progress, Lambdatron already supports a number of useful features. Primitives include booleans, strings, integers, and floating-point values, with more to come. Supported collections include lists, vectors, and hashmaps. Functions are first-class citizens, and when defined can capture values in their environment as closures. Macros are supported, as are syntax-quote, unquote, and unquote-splice. `recur` can be used with `loop` to perform iteration, or with functions to perform tail-call optimized recursion. `let` can be used to define lexically scoped local bindings. A growing standard library and collection of built-in functions provides important core functionality.

Source code and detailed instructions can be found [at the project's GitHub page][lbt-link]. Contributions and feedback are welcome!


## Hakawai ##

**Hakawai** is a iOS library implemented in Objective-C that provides a more powerful `UITextView` meant for applications where lightweight but powerful text editing capabilities are necessary. It exposes a number of additional convenience APIs, such as block-based transformers for manipulating the text view's text, as well as a plug-in system that allows the text view's functionality to be extended in an orderly fashion.

Hakawai's most powerful feature is an included plug-in enabling the creation of social media 'mentions', or text annotations which users create by selecting from a list of options. For example, users can 'mention' other LinkedIn members or companies by typing an '@' symbol. The mentions plug-in is fully-featured, configurable, and generic enough to be used in almost any application. A simple three-function delegate API is all the plug-in requires from the host app - one method to ask the delegate to turn a search query into candidate entities, and two methods to ask the delegate to provide a table view cell for a given entity.

Check it out [here][hkw-link], and read the LinkedIn engineering blog post about it [here][hkw-blog-link].


[lbt-link]:       https://github.com/austinzheng/swift-lambdatron/
[hkw-link]:       https://github.com/linkedin/Hakawai
[hkw-blog-link]:  http://engineering.linkedin.com/ios/introducing-hakawai-powerful-mentions-enabled-text-view-ios
