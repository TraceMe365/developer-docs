---
title: Partial Template Caching
summary: Cache a section of a template Reduce rendering time with cached templates and understand the limitations of the ViewableData object caching.
icon: tags
---

# Partial template caching

Partial template caching is a feature that allows caching of rendered portions of templates. Cached content
is fetched from a [cache backend](../performance/caching), instead of being regenerated for every request.

## Base syntax

```ss
<% cached $CacheKey if $CacheCondition %>
  $CacheableContent
<% end_cached %>
```

This is not a definitive example of the syntax, but it shows the most common use case.

[note]
See also the [Complete syntax definition](#complete-syntax-definition) section
[/note]

The key parts are `$CacheKey` and `$CacheCondition`.
The following sections explain every one of them in more detail.

`$CacheableContent` is just any regular template content, including raw markup, templating syntax, etc.

### `$CacheKey`

Defines a unique key for the cache storage.

[warning]
Avoid heavy computations in `$CacheKey` as it is evaluated for every template render.
[/warning]

The syntax is an optional list of template expressions delimited by commas:

```ss
<% cached [$key1[, $key2[, ...[, $keyN]]]] ... %>

<%-- e.g: --%>
<% cached $UniqueKey, $CurrentMember.ID, $MyHasOne.ID %>
```

The final value is concatenated by the Template Engine into a string. When doing so, Template Engine
adds some extra values to the mix to make it more unique and prevent clashing between cache keys from
different templates.

Here is how it works in detail:

1. `SilverStripe\View\SSViewer::$global_key` hash

   With the current template context, value of the `$global_key` variable is rendered into a string and hashed.

   `$global_key` content is inserted into the template "as is" at the compilation stage. Changing its value
   won't have any effect until template recompilation (e.g. on cache flush).

   By default it equals to `'$CurrentReadingMode, $CurrentUser.ID'`.
   This ensures the current [Versioned](api:SilverStripe\Versioned\Versioned) state and user ID are used.
   At runtime that will become something like `'LIVE, 0'` (for unauthenticated users in live mode).

   As usual, you may override its value via YAML configs. For example:

   ```yml
   # app/_config/view.yml
   SilverStripe\View\SSViewer:
     global_key: '$CurrentReadingMode, $CurrentUser.ID, $CurrentLocale'
   ```

1. Block hash

   Everything between the `<% cached %> ... <% end_cached %>` is taken as text (with no rendering) and hashed.

   This is done at the template compilation stage, so
   the compiled version of the template contains the hash precalculated.

   The main purpose of the block hash is to invalidate cache when the template itself changes.

1. `$CacheKey` hash

   All keys of `$CacheKey` are processed, concatenated and the final value is hashed.
   If there are no values defined, this step is skipped.

1. Make the final key value

   A string produced by concatenation of all the values mentioned above is used as the final value.

   Even if `$CacheKey` is omitted, `SilverStripe\View\SSViewer::$global_key` and `Block hash` values are still
   getting used to generate cache key for the caching backend storage.

#### Cache key calculated in controller

If your caching logic is complex or re-usable, you can define a method on your controller to generate a cache key
fragment.

For example, a block that shows a collection of rotating slides needs to update whenever the relationship
`Page::$many_many = ['Slides' => 'Slide']` changes. In `PageController`:

```php
namespace App\PageType;

use PageController;

class MyPageController extends PageController
{
    // ...

    public function SliderCacheKey()
    {
        $fragments = [
            'Page-Slides',
            $this->ID,
            // identify which objects are in the list and their sort order
            implode('-', $this->Slides()->Column('ID')),
            // identify if any objects are updated - works for both has_many and many_many relationships
            $this->Slides()->max('LastEdited'),
        ];
        return implode('-_-', $fragments);
    }
}
```

Then reference that function in the cache key:

```ss
<% cached $SliderCacheKey %>
```

### `$CacheCondition`

Defines if caching is required for the block.

The condition is optional and if omitted, `true` is implied.

If the condition resolves to `false`, the block skips `$CacheKey` evaluation completely, does not lookup
the data in the cache storage, neither preserve any data in the storage.
The template within the block keeps working as is, same as it would do without
`<% cached %>` block surrounding it.

Although `$CacheCondition` is optional, it is highly recommended. For example,
if you use `$ID` as your `$CacheKey`, you may use `$ID > 0` (or simply `$isInDB`) as the condition.

```ss
<% cached $ID if $ID > 0 %>
```

Without a condition:

- your cache backend will always be queried for cache (for every template render)
- your cache backend may be cluttered with redundant and useless data

[warning]
The `$CacheCondition` value is evaluated on every template render and should be as lightweight as possible.
If you need a complex condition, it may be sensible to calculate the condition in `onBeforeWrite()` for your model and store the result in the database.
[/warning]

## Cache storage

The cache storage may be re-configured via `Psr\SimpleCache\CacheInterface.cacheblock` key for [Injector](../extending/injector).
By default, it is initialised by `SilverStripe\Core\Cache\DefaultCacheFactory` with the following parameters:

- `namespace: "cacheblock"`
- `defaultLifetime: 600`

[note]
The defaultLifetime is in seconds, so a value of `600` means every cache record expires in 10 minutes.
If you have good `$CacheKey` and `$CacheCondition` implementations, you may want to tune these settings to
improve performance.
[/note]

Example below shows how to set partial cache expiry to one hour.

```yml
# app/_config/cache.yml
---
Name: app-cache
After:
  - 'corecache'
---
SilverStripe\Core\Injector\Injector:
  Psr\SimpleCache\CacheInterface.cacheblock:
    constructor:
      defaultLifetime: 3600
```

See [Execution pipeline: Manifests](/developer_guides/execution_pipeline/manifests) for information about how compiled templates are stored.

## Nested cached blocks

Every nested cache block is processed independently.

Let's consider the following example:

```ss
<% cached $PageKey %>
  <%-- Header goes here --%>

  <% cached $BodyKey %>
    <%-- Body goes here --%>
  <% end_cached %>

  <%-- Footer goes here --%>
<% end_cached %>
```

The template processor will transparently flatten the structure into something similar to the following pseudo-code:

```ss
<% cached $PageKey %><%-- Header goes here --%><% end_cached %>
<% cached $BodyKey %><%-- Body goes here --%><% end_cached %>
<% cached $PageKey %><%-- Footer goes here --%><% end_cached %>
```

[note]
`$PageKey` is used twice, but evaluated only once per render because of [template object caching](caching/#object-caching).
If the body section should also be cached with the same requirements as the header and footer sections, it may make sense to use `<% cached $PageKey, $BodyKey %>`.
[/note]

## Uncached

The tag `<% uncached %> ... <% end_uncached %>` disables caching for its content.
In this example, the body content is not cached, but the header and footer sections are.

```ss
<% cached $PageKey %>
  <%-- Header goes here --%>

  <% uncached %>
    <%-- Body goes here --%>
  <% end_uncached %>

  <%-- Footer goes here --%>
<% end_cached %>
```

Because of the nested block flattening (see above), it works seamlessly on any level of depth.

[warning]
The `uncached` block only works on the lexical level.
If you have a template that caches content rendering another template with included uncached blocks,
those will not have any effect on the parent template caching blocks.
[/warning]

## Nesting in LOOP and IF blocks

Currently, a cache block cannot be included in `if` and `loop` blocks.
The template engine will throw an error letting you know if you've done this.

[note]
You may often get around this using aggregates or by un-nesting the block.

For example:

```ss
<% cached $LastEdited %>
  <% loop $Children %>
      <% cached $Up.LastEdited, $LastEdited %>
          $Name
      <% end_cached %>
  <% end_loop %>
<% end_cached %>
```

Might be re-written (and more efficient) as something like this:

```ss
<% cached $LastEdited %>
    <% cached $LastEdited, $AllChildren.max('LastEdited') %>
        <% loop $Children %>
            $Name
        <% end_loop %>
    <% end_cached %>
<% end_cached %>
```

[/note]

## Unless (syntax sugar)

The `if` keyword may be swapped with the `unless` keyword, which inverts the boolean value evaluation.

The two following forms produce the same result

```ss
<% cached unless $Key %>
  "unless $Cond" === "if not $Cond"
<% end_cached %>
```

```ss
<% cached if not $Key %>
  "unless $Cond" === "if not $Cond"
<% end_cached %>
```

## Complete syntax definition

```ss
<% [un]cached [$CacheKey[, ...]] [(if|unless) $CacheCondition] %>
  $CacheContent
<% end_[un]cached %>
```

## Examples

```ss
<% cached %>
  The key is: hash of the template code within the block with $global_key.
  This content is always cached.
<% end_cache %>
```

```ss
<% cached $Key %>
    Cached separately for every distinct $Key value
<% end_cached %>
```

```ss
<% cached $KeyA, $KeyB %>
    Cached separately for every combination of $KeyA and $KeyB
<% end_cached %>
```

```ss
<% cached $Key if $Cond %>
    Cached only if $Cond == true
<% end_cached %>
```

```ss
<% cached $Key unless $Cond %>
    Cached only if $Cond == false
<% end_cached %>
```

```ss
<% cached $Key if not $Cond %>
    Cached only if $Cond == false
<% end_cached %>
```

```ss
<% cached 'contentblock', $LastEdited, $CurrentMember.ID if $CurrentMember && not $CurrentMember.isAdmin %>
  <%--
       Hash of this content block is also included
       into the final Cache Key value along with
       SilverStripe\View\SSViewer::$global_key
  --%>
  <% uncached %>
      This text is always dynamic (never cached)
  <% end_uncached %>
  <%-- This bit is cached again --%>
<% end_cached %>
```
