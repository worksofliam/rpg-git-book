# Introduction

## What is Git

Git is a source change management tool, or version control software, used to keep track of changes to your source code.  
  
## Don't we already have source change management (SCM) products on IBM i?  

There have been, and still are, many SCM products available on IBM i: Turnover, Implementer, TD/OMS, ARCAD, MDCMS, and others.  
  
## How is Git different?  

Git is free and open source!  

Git is the most widely used version control system (it's estimated that 90% or more of developers/projects are using Git).  There are many tutorials, and lots of documentation, available for Git. It's likely that new developers will already understand how to use Git.

Git is a Distributed Version Control System.  When using Git you don't "check out" a single source file, you download (or "clone") the entire repository; you get all the files and all the history for the entire project.  The fact that everyone gets a full copy of the repository allows for concurrent development; in fact, you can be working on multiple different versions of the same source file concurrently!

Branches are really cool.

## Is Git up to the task? 

Git was created by the Linux development team, and they have been using Git for development of the Linux operating system for many years.  In addition to the Linux project, many large projects and companies use Git:
   * Google
   * Facebook
   * Microsoft
   * Twitter
   * Netflix
   * NodeJS
   * Android

## RPG in Git 

Git can easily be used for you ILE source files, as you will see in this guide.  However, because Git was built by the open source community to track their open source stream files, there are a few things to keep in mind when using Git on IBM i:
   * Git does not understand libraries or source physical files.
   * Git doesn't compile your source code or create any objects.
   * Git does not "promote" or distribute objects.  

All of these items will be covered and dealt with through the rest of this guide! 