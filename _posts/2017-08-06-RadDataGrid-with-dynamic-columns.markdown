---
layout: post
title: UWP RadDataGrid with dynamically generated columns
author: Andrii Snihyr
date: '2017-08-06T17:15:00.001-07:00'
tags:
- Telerik
- DataGrid
- RadDataGrid
- Columns
- Dynamic
- C#
- UWP
- VSTS API
- MVVM
- attached property
modified_time: '2017-08-06T17:15:52.560-07:00'
---
Recently I got a task to create a reporting tool that will aggregate data from VSTS and provide reports on the team performance in terms of finished work items and pull requests. I was excited about this, because I saw this as an opportunity to use [Telerik's data grid](http://docs.telerik.com/windows-universal/controls/raddatagrid/overview) in the real world scenario.
<!--more-->

The [VSTS API](https://www.visualstudio.com/en-us/docs/integrate/api/wit/work-items) for work items returns field values as dictionary, like this:
```json
"fields": {
        "System.AreaPath": "Fabrikam-Fiber-Git",
        "System.TeamProject": "Fabrikam-Fiber-Git",
        "System.IterationPath": "Fabrikam-Fiber-Git",
        "System.WorkItemType": "Product Backlog Item",
        "System.State": "New",
        "System.Reason": "New backlog item",
        "System.CreatedDate": "2014-12-29T20:49:20.77Z",
        "System.CreatedBy": "Jamal Hartnett <fabrikamfiber4@hotmail.com>",
        "System.ChangedDate": "2014-12-29T20:49:20.77Z",
        "System.ChangedBy": "Jamal Hartnett <fabrikamfiber4@hotmail.com>",
        "System.Title": "Customer can sign in using their Microsoft Account",
        "Microsoft.VSTS.Scheduling.Effort": 8,
        "WEF_6CB513B6E70E43499D9FC94E5BBFB784_Kanban.Column": "New",
        "System.Description": "Our authorization logic needs to allow for users with Microsoft accounts (formerly Live Ids) - http://msdn.microsoft.com/en-us/library/live/hh826547.aspx"
      },
      "url": "https://fabrikam-fiber-inc.visualstudio.com/DefaultCollection/_apis/wit/workItems/297"
    }
```

RadDataGrid can generate columns out of that, but in my case I wanted to specify exactly what columns will be on the view. Easy I though, there is a "Columns" property and I'll just bind it to the list of columns in my view-model. That is where I was surprised to learn that I can't bind to the "Columns" property of the grid as it is read-only property. 

![FileNotFoundException]({{ "images/RadDataGrid-with-dynamic-columns/ColumnsReadOnly.png" | relative_url }} "RadDataGrid columns property is read-only")

Since I like my UWP apps to follow MVVM pattern and I don't like to have stuff in my code-behind files, exposing grid via interface to my view-model was not an option. I needed a way to bind my view-model property that holds a list of columns to my grid.
That's when I remembered about [Attached Properties](https://docs.microsoft.com/en-us/windows/uwp/xaml-platform/custom-attached-properties). The attached property does exactly what I need, it allows you to add a property that you can bind to.
Here is how it looks:

```csharp
public static readonly DependencyProperty ColumnsSourceProperty = DependencyProperty.RegisterAttached("ColumnsSource", typeof(object),
            typeof(RadGridDynamicColumns), new PropertyMetadata(null, ColumnsSourceChanged));

public static object GetColumnsSource(DependencyObject obj)
{
    return obj.GetValue(ColumnsSourceProperty);
}

public static void SetColumnsSource(DependencyObject obj, object value)
{
    obj.SetValue(ColumnsSourceProperty, value);
}

private static void ColumnsSourceChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
{
    var grid = d as RadDataGrid;
    var query = e.NewValue as WorkItemsQuery;
    if (grid == null || query == null)
        return;
    grid.ItemsSource = null;
    grid.Columns.Clear();
    foreach (var column in query.Columns)
    {
        grid.Columns.Add(new DataGridTextColumn() { PropertyName = column.ReferenceName, Name = column.Name, Header = column.Name });
    }
}
```
All the magic happens in `ColumnsSourceChanged` method. In there I'm iterating over my "Columns" property and add columns to the grid one by one. Before doing that it is important to clear both existing columns and "ItemSource", if I don't do that I'll get duplicated set of columns or an exception when trying to add new columns.

Here is how it looks in XAML:

```xml
<grid:RadDataGrid RelativePanel.Below="QuerySelector"
                  x:Name="WorkItemsGrid"
                  ItemsSource="{x:Bind VM.WorkItems, Mode=OneWay}" 
                  AutoGenerateColumns="False"
                  customizations:RadGridDynamicColumns.ColumnsSource="{x:Bind VM.SelectedQuery, Mode=OneWay}"
                  ColumnResizeHandleDisplayMode="Always"
                  ColumnDataOperationsMode="Inline">
```
Nice and easy, but most importantly code-behind stays empty :)

Happy coding!