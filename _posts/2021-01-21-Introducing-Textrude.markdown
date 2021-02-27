---
layout: post
title:  "Introducing Textrude"
description: "using Scriban to generate code from data"
comments: true
date:   2021-01-21
categories: textrude tools
---

If you've been a developer for a while you've probably used some form of code-generation technology. Maybe you needed to create some API documentation with SWIG, built some C# classes with T4, or lashed together some Python to turn the nth iteration of that list of marketing-approved error messages into string constants.


![TextrudeInteractive](/post_images/IntroducingTextrude/TextrudeInteractive.png)

Generally you need to take some structured data in a human-friendly format such as CSV or JSON and apply a mixture of code and boilerplate text to spit out new files in your language of choice which you can then feed to the compiler.

I've yet to find a tool that provides both a simple internal model representation and a flexible templating language, let alone one that supports code re-use and fits nicely within a typical build system. Textrude is my attempt to provide an easy "on ramp" for simple code-generation tasks: it makes it easy to generate lookup tables, strong-enums, dictionaries or to transform raw data into a form that makes it usable at compile-time.

## Textrude features

 - turns CSV, JSON or YAML input into an understandable and accessible data model operates on that model via Scriban templates which support both in-line text boilerplate and logical constructs
 - adds environment variables and user-supplied definitions to the data-model to make it easier to stamp generated code with build information
- solves the "trailing comma" problem using Scriban's for.last variable
- allows a single template to combine multiple models and to produce multiple output files (for example, generate a .h header file and .cpp source file from the same template)
- contains its own dependency-checking mechanism that allows you to avoid rebuilding output files unless the input has changed.
- is provided as a cross-platform CLI executable
- comes with a GUI prototyping tool that shows you what the output will look like in real-time as you edit input.
- exposes a number of useful functions that make it easy to apply naming-conventions to output code
- is free and open-source

## A quick example

Let's suppose marketing have sent you a list of messages to show the user when your product doesn't work as intended. They've provided this as a CSV because, to most of the organisation, "everything is a spreadsheet"….

Id|Text
--|----
disk error | hard-disk is faulty - save your work at once!
keyboard stuck | press F1 to reboot
faulty touchscreen | please recalibrate the display


<br/>
You want to turn this into code like this….

```cpp
/* Built 18 Jan 2021 on machine BUILDSERVER_5 */
public static class SystemErrors
{

 public const string DiskError = 
 "hard-disk is faulty - save your work at once!";
 
 public const string KeyboardStuck = 
 "press F1 to reboot";
 
 public const string FaultyTouchscreen = 
 "please recalibrate the display";
```
It's easy to do this with a simple template file in Textrude…

```liquid
/* Built {{date.now}} on {{env.COMPUTERNAME}} */
public static class SystemErrors
{
 {{for row in model}}
 public const string {{row.Id | humanizr.pascalize}} = 
 "{{row.Text | string.strip}}";
 {{end}} 
}
```


Things to note:
- There's no need to do anything to process the CSV file; it's automatically turned into a data model for you. Columns in the spreadsheet appear as properties which you can "dot into" and the model itself is just an enumerable array of rows
- Double-braces are used to mark Scriban code blocks with boilerplate text interspersed
- Contextual information such the time and environment information is made available automatically
- Built-in helpers such as the Humanizer "pascalize" method can be used to provide the desired casing. Scriban uses the "pipe" operator to chain function calls

## TextrudeInteractive for quick prototyping

One of the frustrating things I've found about code-generation is the length of the feedback cycle. Typically you need edit your template, run some scripts. load the output into a text editor, and then look for problems. If you feed faulty input into the compiler, it's not uncommon to see several thousand build errors!

The "other half" of Textrude is TextrudeInteractive; a Windows UI that gives you feedback with every keystroke. If you want to try the example above, just paste the CSV file into the "model" window (and select "CSV" as the model type) then paste the template into the "template" window. You should immediately see the results in the output window. If you start editing either the model or the template you should see the output window reflecting those changes in real-time.

## Download
If you're interested in trying out Textrude you can download the source and prebuilt binaries from [github](https://github.com/NeilMacMullen/Textrude)