# Introduction

This manual is a guide to the Dhall configuration language, a programmable file format you can use to simplify large and unwieldy configuration files.

This book addresses readers familiar with Dhall's basic features and interested in using the language within larger projects while following best practices.  In particular, this manual is not a tutorial, meaning that you might not find the contents helpful if you have never used Dhall before.  If you prefer introductory material then you might visit [the official website](http://dhall-lang.org/) which explains the rationale behind the language and links to tutorials more suitable for newcomers.

Inside you will find a collection of "How to" guides for common tasks you will encounter when using Dhall "in anger".  These guides come in two parts:

* **Part 1** - Application authors

  This part covers guides relevant to an "end user" of the language using Dhall to configure a project.  Most people reading this book will fall into this category.

* **Part 2** - Package authors

  This part covers guides relevant to contributors who wish to share reusable Dhall packages with others.  You might still find this section interesting even if you don't plan to share code, if only to recognize common packaging idioms.

Each chapter is written to be as self-contained as possible, yet they will build upon a running example so the book can be read front to back.

Reading this manual will give you the confidence and ability to share the Dhall configuration language with your colleagues.  Does a new person need assistance setting up their development environment to use Dhall?  Is somebody stuck on a tricky type error?  Do several contributors need to agree upon an opinionated project layout?  There's a ready-to-share chapter for each of those subjects (and more) that you can recommend to others.

You can trust that the advice in this book reflects best practices because I've spent years using the language, supporting other users (both open source and commercial), and steering the ecosystem as the original author of the language.

I assume that you care about correctness and quality if you've gotten this far, so I provide this personal guarantee: I will refund your purchase of this book, no questions asked, if you believe your use of Dhall has not noticeably reduced your project's defect rate.  So continue reading if you want to deploy with confidence and build upon a software foundation that you can trust.