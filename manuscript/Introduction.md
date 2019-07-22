# Introduction

This manual is a guide to the Dhall configuration language, a programmable file
format commonly used to tame large and unwieldy configurations.

You should read this manual if you are interested in transitioning from a novice
user of the Dhall configuration language to a proficient user.  In other words,
I dedicate this book to readers familiar with basic language features and
interested in using Dhall within larger projects while following best practices.

This manual is not a tutorial, meaning that you probably will not find the
contents interesting if you have never used Dhall before.  If that describes you
then you might prefer [the official website](http://dhall-lang.org/) which
explains the rationale behind the language and links to tutorials more suitable
for newcomers.

Inside you will find a collection of "How to" guides providing detailed recipes
for common tasks you will encounter when using Dhall "in anger".  Each section
is written to be as self-contained as possible, yet each will build upon a
running worked example to provide logical continuity so the book can be read
front to back.

These sections come in two parts:

* **Part 1** - Application authors

  This part covers guides relevant to an "end user" of the language using Dhall
  to configure a project.  Most people reading this book will fall into this
  category.

* **Part 2** - Package authors

  This part covers guides relevant to contributors who wish to share reusable
  Dhall packages with others.  You might still find this section interesting
  even if you don't plan to share code, if only to recognize idioms that package
  authors will adhere to.

Reading this manual will give you the confidence and ability to share the Dhall
configuration language with your colleagues.  Does a new person need assistance
setting up their development environment to use Dhall?  Is somebody stuck
working through a gnarly type error?  Do several contributors need to agree upon
an opinionated project layout?  There's a ready-to-share chapter for each of
those subjects (and more) that you can recommend to others.

You can trust that the advice in this book reflects best practices because I've
spent years using the language, supporting other users (both open source and
commercial), and steering the ecosystem as the original author of the language.

I assume that you care about correctness and quality if you've gotten this far,
so I provide this personal guarantee: I will refund your purchase of this book,
no questions asked, if you believe your use of Dhall has not noticeably reduced
your project's defect rate.  So continue reading to take back your nights and
weekends by deploying software that you can trust.
