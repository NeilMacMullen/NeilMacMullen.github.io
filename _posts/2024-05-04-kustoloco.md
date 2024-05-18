---
layout: post
title:  "Kusto-Loco"
description: "Simple querying and rendering of in-app data"
comments: false
date:   2024-05-04
categories: kusto
---

## What's KQL?

[Kusto](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/) (KQL) is a query language introduced by Microsoft that is used to interrogate data in Application Insights and [Azure Data Explorer](https://dataexplorer.azure.com/).   It's got a clean syntax and an extensive set of operators and functions.  

<figure>
  <img src="{{site.url}}/post_images/kustoloco/sample1.png" alt="A sample Kusto query"/>
  <figcaption>A typical Kusto query</figcaption>
</figure>
<br/>
That's great if you're looking at Application Insights logs or you've had the foresight to store your data in an ADX cluster but what if you've written a protocol analyser and want to let power-users write those kind of queries against data they've just collected?

## Introducing Kusto-Loco

[Kusto-Loco](https://github.com/NeilMacMullen/kusto-loco) is an open-source implementation of the Kusto engine that allows you to easily add query capbility to a dotnet application.  Typically this can be done with just a few lines of code and the engine is performant enough to deal with datasets consisting of tens of millions of rows of data on a typical laptop.

<figure>
  <img src="{{site.url}}/post_images/kustoloco/processeswpf.png" alt="Process queries"/>
  <figcaption>A basic process watcher with query support</figcaption>
</figure>
<br/>
In the rest of the article I'll show you how to build a simple console application that uses Kusto-Loco to query the process-list on your PC.

## KustoQueryContext - a place to hold your data


Just like a traditional database, Kusto uses the concept of named *tables*. You issue queries on an individual table or perform *join* operations on multiple tables to achieve more complicated results.  KustoLoco uses a `KustoQueryContext` object to store the tables availble for queries.   

Tables can be added by loading from a file (CSV, TSV,JSON and Parquet are currently supported) but the easiest way is just to drop a collection of in-memory POCOs into the context via the `WrapDataIntoTable` or `CopyDataIntoTable` methods.

`WrapDataIntoTable` is more efficient for large collections but requires some guarantees about collection immutability so we'll use the `CopyDataIntoTable` method here since the collection is small and there's no appreciable performance loss.

{% highlight csharp %}
var processes = Process
                .GetProcesses()
                .Select(p => new { Name=p.ProcessName, ThreadCount= p.Threads.Count})
                .ToArray();

var context = new KustoQueryContext();
context.CopyDataIntoTable("p", processes);
{% endhighlight %}

Now we have a table called *p* in the context and we're ready to issue queries against it!

## Issuing the query and handling the result

Here's the basic shape of the application: repeatedly read a query from the console, issue it against the context, and do ...*something*... with the result....

{% highlight csharp %}
do
{
    var query = Console.ReadLine();
    var result = await context.RunQuery(query);
    //... do *something* with the result of the query
} while (true);
{% endhighlight %}

So what is the *something* ?  It's not just a matter of displaying a subset of the records we passed in; since KQL has the ability to change the 'shape' of the data by using operators such as *summarize*, *extend*, or *project*.  Instead, the result is a `KustoQueryResult` object which is a wrapper around the returned data that allows us to iterate over it in convenient ways for further processing or rendering.

If we were simply going to dump the result into a WPF DataGrid we could just use `ToTableOrError`...

{% highlight csharp %}
myDataGrid.ItemsSource = result.ToTableOrError().DefaultView; 
{% endhighlight %}

If we knew the _shape_ of the result we could generate a set of POCOs...

{% highlight csharp %}
var pocos = result.ToRecords<ExpectedRecord>();
{% endhighlight %}

If we wanted to send the result across the wire as Json we could use the `AsOrderedDictionarySet` or `ToJson` methods... 

{% highlight csharp %}
var serializableArrayOfObjectsForInsertionToDto = result.AsOrderedDictionarySet();
var jsonText = result.ToJson();
{% endhighlight %}


However, since this in a console application and we want to support arbitrary queries we'll use the wonderful [Spectre.Console](https://spectreconsole.net/widgets/table) library to render a nice table...

{% highlight csharp %}
    var table = new Table();
    // Add each column to the table with the header showing name and type
    foreach (var column in result.ColumnDefinitions()) 
        table.AddColumn($"{column.Name}:{column.UnderlyingType.Name}");

    // Add each row .
    foreach (var row in result.EnumerateRows())
    {
        var rowCells = row.Select(RenderCell);
        table.AddRow([..rowCells]);
        string RenderCell(object? cell) 
               => cell?.ToString() ?? "<no data>";
    }
    // Render the table to the console
    AnsiConsole.Write(table);
{% endhighlight %}

The key thing here is that the `KustoQueryResult` object allows us to traverse the result in either column-wise or row-wise form to better match the expectations of the receiving code.  In this case we need to first set up a set of columns in the Spectre table then add a series of rows.

Note that Kusto represents *missing* data as *null* (except in the case of strings where the empty-string is used).  Even if we're sure that there won't be any *nulls* in the source table, it's possible to construct queries that would emit them in the output so enumerating over the cells in a `KustoQueryResult` yields a set of *nullable* objects.

## Handling Errors

KQL is quite easy to write but what happens if there is a typo or syntax error in the query?  In this case, when we enumerate over the columns and rows we'll get empty sets so it should be "safe" to ignore this case but if we want to provide more useful feedback to the user we can examine the `Error` property 

{% highlight csharp %}
    AnsiConsole.MarkupLineInterpolated(
        result.Error != ""
        ? (FormattableString)$"[red]{result.Error}[/]"
        : $"[green]{result.RowCount} results in {(int)result.QueryDuration.TotalMilliseconds}ms[/]");
{% endhighlight %}

## The finished result

Here's the complete source.  Most of it is just dealing with fetching the data and then rendering the query result !

{% highlight csharp %}
using System.Diagnostics;
using KustoLoco.Core;
using Spectre.Console;

var processes = Process
                .GetProcesses()
                .Select(p => new { Name=p.ProcessName, ThreadCount= p.Threads.Count})
                .ToArray();

var context = new KustoQueryContext();
context.CopyDataIntoTable("p", processes);

do
{
    var query = Console.ReadLine();
    var result = await context.RunQuery(query);

    var table = new Table();
    // Add each column to the table with the header showing name and type
    foreach (var column in result.ColumnDefinitions()) 
        table.AddColumn($"{column.Name}:{column.UnderlyingType.Name}");

    // Add each row .
    foreach (var row in result.EnumerateRows())
    {
        var rowCells = row.Select(RenderCell);
        table.AddRow([..rowCells]);
        string RenderCell(object? cell) => cell?.ToString() ?? "<no data>";
    }
    // Render the table to the console
    AnsiConsole.Write(table);

    AnsiConsole.MarkupLineInterpolated(result.Error != ""
        ? (FormattableString)$"[red]{result.Error}[/]"
        : $"[green]{result.RowCount} results in {(int)result.QueryDuration.TotalMilliseconds}ms[/]");
} while (true);
{% endhighlight %}

<figure>
  <img src="{{site.url}}/post_images/kustoloco/demo.png" alt="output from demo app"/>
  <figcaption>Query output rendered as a Spectre table</figcaption>
</figure>
<br/>
## Rendering charts

Let's be honest, it's usually much nicer to look at charts than tables and KQL has a *render* operator.  Could we support that? 
The answer is...yes! KustoLoco supports rendering via the [Vega-Lite](https://vega.github.io/vega-lite/) charting library.  The **KustoLoco.Rendering** package supports this via a simple method call.

{% highlight csharp %}
KustoResultRenderer.RenderChartInBrowser(result);
{% endhighlight %}

If the user has specified a *render* step in the query, this method will generate HTML, save it to a temporary file, and open the user's browser to render it.   If you're writing a UI based application and want in-app rendering you can use the `RenderToHtml` call to obtain the raw HTML to pass on to a `WebView` control or similar.


Let's see what happens if we add this to the application and append a *render* operator...

<figure>
  <img src="{{site.url}}/post_images/kustoloco/chart.png" alt="output from demo app with render operator"/>
  <figcaption>Rendering a piechart of threadiest applications</figcaption>
</figure>
<br/>
## But wait... there's more!

This article is too long already but there's a lot more to Kusto-Loco than just the engine....

- The KustoLoco.FileFormats package allows serialization from and to CSV,TSV, Parquet and JSON files.  Each of these serializers is an instance of the `ITableSerializer` which allows you to plug in support for other file formats.

- The project includes a graphical [data-explorer](https://github.com/NeilMacMullen/kusto-loco/wiki/LokqlDX) for quick querying and exploration of data files as well as a CLI version for automated data processing.

- There's even a Powershell module that allows querying of object pipelines! 

More about these another time....

## Attribution

[Kusto-Loco](https://github.com/NeilMacMullen/kusto-loco) is based heavily on the [BabyKusto](https://github.com/davidnx/baby-kusto-csharp) codebase created by [davidnx](https://github.com/davidnx),[David Nissimoff](https://github.com/davidni) and [Vicky Li](https://github.com/VickyLi2021) which appears to have been created as part of a Microsoft hackathon.  
