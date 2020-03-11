# Logging in Android

Logging is a useful and often necessary step in developing an Android application. It allows us to log errors, debug messages and other useful information. No doubt, every developer utilizes it one way or another - be it printing messages in console, or setting up an elaborate logging system. 

Although logging may seem like a trivial task, it still can impose several challenges, especially in a multi-module project. In this post we will look at how we can easily set up logging in the app.

## Defining the task

First, let's define the task at hand. Suppose we have a multi-module Kotlin project with this basic structure
