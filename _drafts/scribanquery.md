---
layout: post
title:  "Embedding Scriban as a query language in your application"
description: "Textrude engine"
comments: false
date:   2021-01-21
categories: draft
---

Scriban is a templating language based on Liquid.  It's typically used for generating markup or simple text but in this article I'll show you how you can use it to query structured data with the help of the Textrude-engine package.

Expression-trees and parsers, member representation


First things first, what does Scriban look like ?

A simple example would be...

That's all very well but how do the values we're inserting get into the template?  Textrude can perform this magic for you

```
var filename =args[0];
var query = args[1];
var format = Textrude.GuessFormatFromExtension(filename);
var text = File.ReadAllText(filename);

var output = ApplicationEngine
            .StandardEmbedded()
            .WithModel("items",text,format)
            .WithTemplate(query)
            .Render()
            .ErrorOrOutput;
Console.WriteLine(output);            
```

If we now run 
```
q.exe data.json 'items | array.filter ($0.population < 100) | textrude.to_json'
```


What's going on here?  The application is setting up a context where the elements in the input file have been imported as an array called "items".  The *query* is filtering that array so that we onlt take records where the population field is small.  Finally, we need to transform the filtered records in the Scriban context to a clean string representation with `textrude.to_table`

The nice thing here is that the query will work regardless of whether our data is in JSON, YAML or CSV format.

## In Memory

That's all well and good but what if we want to query in-memory objects?  You may well have written an application to allow the user to query items from a database. The typical way to do this is to construct an Expression Tree - but that's quite a complex approach !  What if we could just type in arbitrarily complicated Scriban queries?  Here's how you can do that...

```

IEnumberable<MyType> objects....
var clause = "population > 1"

var query = "items | array.filter  @queruFunc" 
var success = ApplicationEngine
            .StandardEmbedded()
            .WithModel("items",objects)
            .WithTemplate(query)
            .Render()
            .TryDeserialise<MyType[]>("filtered",out var filteredThings);


```

## TODO - need an import mechanism OR - import as scriptobject not Dictionary!

## Limitations

Objects must be JsonSerialisable.  Custom Fields
Obviously the filtered list is a _copy_ you could just return an array of ids


Library helpers - topic for another day