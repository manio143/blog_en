---
title: 5. TDD with F#
images: []
---

# 5. TDD with F#

_⚠️NOTE: this post in unfinished, but I decided to push it, so I don't loose it._

I'm gonna raise my hand and admit, most of my open source projects so far have poor testing. So to force myself to remedy this I will try to push myself to use Test Driven Development for Excos.

How will this look like? The vast majority of production code should be written in response to **first** writing a test which will describe the desired properties of the system. The tests will operate on the similar level to any potential consumers of the system. This means the tests should either target the API of the web service OR the user facing website. For components which have a very high number of pathways to test but a single API entrypoint I might go down a level with the tests to improve their performance.

I recommend watching this talk: [Building Operable Software with TDD (but not the way you think) - Martin Thwaites - NDC Porto 2024](https://www.youtube.com/watch?v=vzr4HiQZhdY).

The general idea is to write feature specs in code - the test file (whether integration test or unit test) would have prose in comments explaining the design reasoning.
This idea came to me by looking at [Literate Programming](https://fsprojects.github.io/FSharp.Formatting/literate.html).
It could be combined with a DSL to make tests more human readable and allow exporting .ts file into HTML for reading the specs.
