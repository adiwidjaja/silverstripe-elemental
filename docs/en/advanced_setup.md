# Elemental - advanced setup

This documentation assumes that the reader is already familiar with basic concepts
of the [Elemental module](https://github.com/dnadesign/silverstripe-elemental) and the [Fluent module](https://github.com/tractorcow-farm/silverstripe-fluent).
This document provides an advanced setup guide for enterprise scale projects using these modules.


## Elemental setup

### Page setup

It's a good idea to create a `BlockPage` class to represent a page with blocks (i.e. avoid adding blocks directly to the `Page`).
This allows more flexibility as other page types can subclass `Page` without inheriting any blocks related functionality.
This is useful for covering edge cases that may appear during projects (i.e. not all pages may need blocks).

```php
class BlockPage extends Page
{
    /**
     * @var array
     */
    private static $extensions = [
        ElementalPageExtension::class,
    ];

    // ...
}
```


### Block setup

It is possible to share blocks between pages, but this may be a little bit tricky when it comes to content editing.
Block should represent a chunk of page content, so editing it should not effect other pages.
This depends on the project, but in most cases the content authors will be working per page (top down),
so sharing blocks is probably something to avoid.
Shared blocks become even less useful when large number of block instances are present as it becomes almost impossible to find the right one.

The overall recommendation is to only allow a content block to be used on only one page.
The main benefit of using blocks is to reuse patterns and functionality across pages, not necessarily content data.
It's possible to add functionality which allows content authors to copy specific blocks to other pages for a quick transfer of content data.


Elemental editor `GridField` needs to be adjusted accordingly:

```php
/**
 * Apply strong inheritance relation config
 * no existing auto complete as reusing items is not allowed
 * no unlink button for the same reason
 *
 * @param GridFieldConfig $config
 * @return GridFieldConfig
 */
public static function strongInheritanceConfig(GridFieldConfig $config): GridFieldConfig
{
    $config->removeComponentsByType([
        // remove add existing button since this is a full ownership relation
        GridFieldAddExistingAutocompleter::class,
        // remove archive action because nested objects are expected to be publish / un-publish via page
        GridFieldArchiveAction::class,
    ]);

    /** @var GridFieldDeleteAction $deleteAction */
    $deleteAction = $config->getComponentByType(GridFieldDeleteAction::class);

    if ($deleteAction !== null) {
        // replace unlink relation button with delete button since this is a full ownership relation
        $deleteAction->setRemoveRelation(false);
    }

    return $config;
}
```

Note that we also want to remove `publish` and `archive` actions from blocks.
These actions will be done only on page level and will cascade down to blocks.
Make sure to properly configure your objects with the `owns` and `cascade_deletes`.


## Elemental with Fluent setup

`Fluent` module provides multiple options how to localise your content, but there is one option which is the best on average: `indirect localisation`.
The only thing that will be localised directly is the `ElementalArea`relation.

For the new Fluent 4.4, the ForeignKey data type has to be allowed to be translated.

```php
class BlockPage extends Page
{
    /**
     * @var array
     */
    private static $field_include = [
        'ElementalAreaID',
    ];

    // For Fluent 4.4
    private static $data_include = [
        'ForeignKey',
    ];

    // ...
}
```

This configuration allows us to have different `ElementalArea` for different locales of the page.
We also need to create a copy of the `ElementalArea` when content is being localised. 

```php
class BlockPage extends Page
{
    public function onBeforeWrite()
    {
        parent::onBeforeWrite();
    
        if (!$this->isDraftedInLocale() && $this->isInDB()) {
            $elementalArea = $this->ElementalArea();
    
            $elementalAreaNew = $elementalArea->duplicate();
            $this->ElementalAreaID = $elementalAreaNew->ID;
        }
    
        return;
    }
    
    // ...
}
```

Note that it's important to have the [cascade_duplicates setting](https://docs.silverstripe.org/en/4/developer_guides/model/relations/#cascading-duplications) present on all the relevant objects so they would copy as well.

Furthermore, we also need to disable the inheritance for blocks.
The Fluent module provides multiple extension points, one of them being the `updateLocaliseSelect`.
We need to create an `Extension` with the following code and apply it to the `BlockPage`:

```php
class BlockPageFluentExtension extends Extension
{
    /**
     * Override default Fluent fallback
     *
     * @param string $query
     * @param string $table
     * @param string $field
     * @param Locale $locale
     */
    public function updateLocaliseSelect(&$query, $table, $field, Locale $locale)
    {
        // disallow elemental data inheritance in the case that published localised page instance already exists
        if ($field == 'ElementalAreaID' && $this->owner->isPublishedInLocale()) {
            $query = '"' . $table . '_Localised_' . $locale->getLocale() . '"."' . $field . '"';
        }
    }
}
```


```php
class BlockPage extends Page
{
    /**
     * @var array
     */
    private static $extensions = [
        ElementalPageExtension::class,
        BlockPageFluentExtension::class,
    ];
    
    // ...
}
```

#### Benefits of indirect localisation

* different localisation of pages can have completely different set of blocks which allows greater flexibility
* localisation is only on page level, so any functionality on block level does not need to care about localisation
* this is especially useful when writing unit tests as it is significantly easier to set up tests without localisation

#### Downsides of indirect localisation

* the blocks are unaware of their locales which makes bottom up relation lookup slower
* this can be remedied by some extra data stored in blocks (see notes below)

### Block performance increase

If only one page can own a block, we can store page reference directly on the block.
This can be done when block is created (`onBeforeWrite`). This is helps significantly when traversing relation from block up to a page.
Note that there may be a lot of objects sitting between block and a page.
For example `Content block -> Elemental area -> Layout block -> Elemental area -> Page`.

Additional information can be stored like page locale (relevant for `Fluent` module) to specify the target data even further.

### Unit tests

Writing unit tests for `Elemental` with `Fluent` can be rather tricky to figure out.
Here are some guidelines to make that easier.

#### Make sure your fixture has some locales setup

It's important to include some locales because otherwise your test might be testing a very different situation.
Example `Locale` setup in a fixture:

```yml
TractorCow\Fluent\Model\Locale:
  nz:
    Locale: en_NZ
    Title: 'English (New Zealand)'
    URLSegment: newzealand
  us:
    Locale: en_US
    Title: 'English (US)'
    URLSegment: usa
    Fallbacks:
      - =>TractorCow\Fluent\Model\Locale.nz
```

#### Localised fixture data (automatic, single locale)

In the case your fixture needs to contain data for only a single locale you can specify your desired locale in your unit test like this:

```php
protected function setUp()
{
    // Set locale for fixture creation
    FluentState::singleton()->withState(function (FluentState $state) {
        $state->setLocale('en_NZ');
        parent::setUp();
    });
}

```

This will localise all your data so you don't need to worry about that in your fixtures. The following fixture will produce a page localised in `en_NZ`:

```yml
App\Pages\OperatorArticlePage:
  article-page1:
    Title: ArticlePage1 NZ
    URLSegment: article-page1
```



#### Localised fixture data (manual, single or multiple locales)

In some cases you want to have multiple locales of one page set up in your fixtures.
This means you need to specify localised data manually.
Example below shows how to specify localised `Title` for two locales of one page.
Note that each localised field has to be specified for the table that actually holds the field.
In this case, it's `SiteTree`.
If you are unsure where your field sits it may be a good idea to check your database structure first and find the relevant table.

```yml
App\Pages\OperatorArticlePage:
  article-page1:
    Title: ArticlePage1
    URLSegment: article-page1

SiteTree_Localised:
  article-page1-nz:
    Locale: en_NZ
    RecordID: =>App\Pages\OperatorArticlePage.article-page1
    Title: ArticlePage1 NZ
  article-page1-au:
    Locale: en_AU
    RecordID: =>App\Pages\OperatorArticlePage.article-page1
    Title: ArticlePage1 AU
```

### Working with Fluent state

Make sure you always use the `FluentState` callback to change the global state like this:

```php
FluentState::singleton()->withState(function (FluentState $state) {
    $state->setLocale('en_NZ');

    // your code goes here
})
```

This is very important as global state is reverted back after the callback is executed so it's safe to be used.
Unit tests benefit mostly from this as this makes sure that there are no dependencies between unit tests as the global state is always changed only locally in one test.
