---
title: Grouping DataObject sets
summary: Learn how to split the results of a query into subgroups
---

# Grouping lists of records

The [SS_List](api:SilverStripe\ORM\SS_List) class is designed to return a flat list of records.
These lists can get quite long, and hard to present on a single list.
[Pagination](/developer_guides/templates/how_tos/pagination) is one way to solve this problem,
by splitting up the list into multiple pages.

In this howto, we present an alternative to pagination:
grouping a list by various criteria, through the [`GroupedList`](api:SilverStripe\ORM\GroupedList) class.
This class is a [`ListDecorator`](api:SilverStripe\ORM\ListDecorator), which means it wraps around a list,
adding new functionality.

It provides a `groupBy()` method, which takes a field name, and breaks up the managed list
into a number of arrays, where each array contains only objects with the same value of that field.
Similarly, the `GroupedBy()` method builds on this and returns the same data in a template-friendly format.

## Grouping sets by first letter

This example deals with breaking up a [`SS_List`](api:SilverStripe\ORM\SS_List) into sub-headings by the first letter.

Let's say you have a set of Module objects, each representing a Silverstripe CMS module, and you want to output a list of
these in alphabetical order, with each letter as a heading; something like the following list:

```text
* B
    * Blog
* C
    * CMS Workflow
    * Custom Translations
* D
    * Database Plumber
    * ...
```

The first step is to set up the basic data model,
along with a method that returns the first letter of the title. This
will be used both for grouping and for the title in the template.

```php
namespace App\Model;

use SilverStripe\ORM\DataObject;

class Module extends DataObject
{
    private static $db = [
        'Title' => 'Text',
    ];

    /**
     * Returns the first letter of the module title, used for grouping.
     * @return string
     */
    public function getTitleFirstLetter()
    {
        return $this->Title[0];
    }
}
```

The next step is to create a method or variable that will contain/return all the objects,
sorted by title. For this example this will be a method on a new `ModulePage` class.

```php
namespace App\PageType;

use App\Model\Module;
use Page;
use SilverStripe\ORM\GroupedList;

class ModulePage extends Page
{
    /**
     * Returns all modules, sorted by their title.
     * @return GroupedList
     */
    public function getGroupedModules()
    {
        return GroupedList::create(Module::get()->sort('Title'));
    }
}
```

[notice]
Notice that we're sorting as part of the ORM call. While `GroupedList` does have a `sort()` method, it doesn't work how you might expect, as it returns a sorted copy of the underlying list rather than sorting the list in place.
[/notice]

The final step is to render this into a template. The `GroupedBy()` method breaks up the set into
a number of sets, grouped by the field that is passed as the parameter.
In this case, the `getTitleFirstLetter()` method defined earlier is used to break them up.

```ss
<%-- Modules list grouped by TitleFirstLetter --%>
<h2>Modules</h2>
<% loop $GroupedModules.GroupedBy("TitleFirstLetter") %>
    <h3>$TitleFirstLetter</h3>
    <ul>
        <% loop $Children %>
            <li>$Title</li>
        <% end_loop %>
    </ul>
<% end_loop %>
```

## Grouping sets by month

Grouping a set by month is a very similar process.
The only difference would be to sort the records by month name, and
then create a method on the DataObject that returns the month name,
and pass that to the [GroupedList::GroupedBy()](api:SilverStripe\ORM\GroupedList::GroupedBy()) call.

We're reusing our example `Module` object,
but grouping by its built-in `Created` property instead,
which is automatically set when the record is first written to the database.
This will have a method which returns the month it was posted in:

```php
namespace App\Model;

use SilverStripe\ORM\DataObject;

class Module extends DataObject
{
    /**
     * Returns the month name this news item was posted in.
     * @return string
     */
    public function getMonthCreated()
    {
        return date('F', strtotime($this->Created));
    }
}
```

The next step is to create a method that will return all records that exist,
sorted by month name from January to December. This can be accomplshed by sorting by the `Created` field:

```php
namespace App\PageType;

use App\Model\Module;
use Page;
use SilverStripe\ORM\GroupedList;

class ModulePage extends Page
{
    /**
     * Returns all news items, sorted by the month they were posted
     * @return GroupedList
     */
    public function getGroupedModulesByDate()
    {
        return GroupedList::create(Module::get()->sort('Created'));
    }
}
```

The final step is to render this into the template using the [GroupedList::GroupedBy()](api:SilverStripe\ORM\GroupedList::GroupedBy()) method.

```ss
<%-- Modules list grouped by the Month Posted --%>
<h2>Modules</h2>
<% loop $GroupedModulesByDate.GroupedBy("MonthCreated") %>
    <h3>$MonthCreated</h3>
    <ul>
        <% loop $Children %>
            <li>$Title ($Created.Nice)</li>
        <% end_loop %>
    </ul>
<% end_loop %>
```

## Related

- [Howto: "Pagination"](/developer_guides/templates/how_tos/pagination)
