---
layout: post
title: "Javascript Import-maps is enabled in Firefox 108"
date: 2022-11-01 00:00:00 -0000
categories: import-maps
---

## Introduction to Import-Maps
### Module Specifier remapping
People familiar with Javascript modules should also know that you could [import
objects from other Javascript modules]. For example:

```
// In a module script.
import moment from "/node_modules/moment/src/moment.js";
```

Notice that after the **'from'** keyword, you need to provide a string literal with either the absolute
path or the relative path of the module script, so the Javascript engine could
know where the module script is located.

But with import-maps, you could just do

```
// In a module script.
import moment from "moment";
```

Notice the path has been replaced by a [Module specifier] called "moment".

But the Javascript engine still needs to know the location of the module script.
This is the part "Import-maps" comes into play.

In the HTML document[^1], you could provide a script tag whose type is "importmap",
with a JSON string, which is a JSON object that maps the module specifier into the URL.

```
<!-- In the HTML document -->
<script type="importmap">
{
  "imports": {
    "moment": "/node_modules/moment/src/moment.js"
  }
}
</script>
```

So when the Javascript engine tries to resolve the *Module Specifier* "moment",
it will check the import-map in the HTML document and try to get the corresponding
URL of the module specifier.

This trick could be extended a little bit with file hashes. For example, sometimes you need to
cache the module scripts, and let's say the file name will be appended with the hash of
the file's content. In the above example, moment.js could become moment-8e0d62a03.js,
where the 8e0d62a03 is the hash number of the content of moment.js. You could use Import-maps
to keep track of the hashed module script.

```
<!--
An import map example to map the module specifier to the actual cached file
in the HTML document
-->
<script type="importmap">
{
  "imports": {
    "moment": "/node_modules/moment/src/moment-8e0d62a03.js"
  }
}
</script>
```

### Prefix remapping via trailing slash '/'
Import-maps also allow you to remap the prefix of the module specifier, provided
that the entry in the Import-map must end with a trailing slash '**/**'.


```
// In the HTML document.
<script type="importmap">
{
  "imports": {
    "app/": "/js/app/"
  }
}
</script>
```

```
// In a Javascript module script.
import foo from "app/foo.js";
```

In this example, There isn't an entry "app/foo.js" in the Import-map. However,
there's an entry "app/"(notice it ends with a slash '**/**'), so the "app/foo.js"
will be resolved to "/js/app/foo.js".

This feature is quite useful when the module contains several sub-modules, or
when you're about to test multiple versions of the external module,

```
<!-- In the HTML document -->
<script type="importmap">
{
  "imports": {
    "feature/": "/js/module/feature/",
    "app/": /js/app@4.0/",
  }
}
</script>
```

### Sub-folders may need different version of the external module.
Import-Maps provide another mapping called "**scopes**". It allows you to use the 
specific mapping table according to the URL of the module script. For example,


```
// In the HTML document.
<script type="importmap">
{
  "scopes": {
    "/foo/": {
      "app.mjs": "/js/app-1.mjs"
    },
    "/bar/": {
      "app.mjs": "/js/app-2.mjs"
    }
  }
}
</script>
```


Here the *scopes* map has two entries:
1. "/foo/" -> *Module specifier map 1*
2. "/bar/" -> *Module specifier map 2*

For the module scripts located in "/foo/", the "app.mjs" will be resolved to "/js/app-1.mjs",
where as for those located in "/bar/", "app.mjs" will be resolved to "js/app-2.mjs".


```
// In /foo/foo.js
import app from "app.mjs"; // Will import "/js/app-1.mjs"
```


```
// In /bar/bar.js
import app from "app.mjs"; // Will import "/js/app-2.mjs"
```

## Explaination in depth
Let's explain the terms first.
The string literal "app.mjs" in above examples is called *[Module Specifier]* in ECMA Script.
And the map which maps "app.mjs" to a URL is called *[Module Specifier Map]*.

An import map is an object with two optional items:
- **_imports_**, which is a *module specifier map*.
- **_scopes_**, which is a map of URLs to *module specifier maps*.

So an import map could be thought as:
- A top-level module specifier map called "**_imports_**"".
- A map of module specifier maps, which is called "**_scopes_**", that could override the top-level module specifier map according to the location of the referrer.

Put it into a graph

```
Module Specifier Map:
  +------------------+-----+
  | Module Specifier | URL |
  +------------------+-----+
  |  ......          | ... |
  +------------------+-----+
```

```
Import Map:
  imports:
    Top-level Module Specifier Map

  scopes:
    +-------+------------------------+
    | URL 1 | Module Specifier Map 1 |
    +-------+------------------------+
    | URL 2 | Module Specifier Map 2 |
    +-------+------------------------+
    | ...   | ...                    |
    +-------+------------------------+
```

### Validation of entries when parsing the import map
The format of the import map text has some requirements:
- A valid JSON string.
- The parsed JSON string must be a JSON object.
- The *imports* and *scopes* must be JSON objects as well.
- The values in *scopes* must be JSON objects, since they should be the type of *Module Specifier Maps*.

Failed to meet anyone of above requirements will fail to parse the import map,
and a **SyntaxError**/**TypeError** will be thrown.[^2]

After the validation of JSON is done, the parsing of the import map will check
whether the values(URLs) in the Module specifier maps are valid.

If there's any invalid URL, the value of the entry in the module specifier map
will be marked as invalid. Later when the Javascript engine is trying to resolve
the module specifier, if the resolution result is the invalid value, the
resolution will fail and throw a TypeError.

```
<!-- In the HTML document -->
<script type="importmap">
{
  "imports": {
    "foo": "INVALID URL"
  }
}
</script>
```

```
// In a Javascript module script.
import("foo").then(() => {
    }).catch(err => {
      // TypeError will be thrown.
    });
```


### Resolution precedence
When the Javascript engine is trying to resolve the module specifier, it will
find out the most specific *Module Specifier Map* to use, depending on the URL
of the referrer.

The precedence order of the *Module Specifier Maps* from high to low is:
1. scopes
2. imports

After the most specific *Module Specifier Map* is determined, then the resolving will
iterate the parsed module specifier map to find out the best match of the module
specifier:
1. The entry whose key equals to the module specifier.
2. The entry whose key has the **longest common prefix** with the module specifier
, provided the key ends with trailing slash '/'.


```
<!-- In the HTML document -->
<script type="importmap">
{
  "imports": {
    "a/": "/js/test/a/",
    "a/b/": "/js/dir/b/"
  }
}
</script>

```

```
// In a Javascript module script.
import foo from "a/b/c.js"; // will import "/js/dir/b/c.js"
```

Notice although the first entry "a/" in the import map could be used to resolve "a/b/c.js",
however there is a better match below "a/b/" since it has a longer common prefix
of the module specifier. So "a/b/c.js" will be resoved to "js/dir/b/c.js", instead
of "/js/test/a/b/c.js".

Details could be found in [resolve a module specifier].

### Limitations of Import-maps
Currently there are some limitations of the Import-maps, but these may be lifted
in the future:
- Only one import-map is supported
  - Processing the first import-map script tag will disallow the following import maps being processed.
    Those import map script tags won't be parsed and the onerror handlers will be called.
    Note that even the first import map is faield to parse, those import maps
    afterwards still won't be processed.
- Not supported for external import-maps. See [issue 235].
- Import-maps won't be processed if the module loading has been started.
- Not supported for workers/worklets. See [issue 2].

## Common problems when using import-maps
There are some common problems when you try to use Import-maps for the first
time:
- Invalid JSON format
Check the 'Validation' part above, if the validation of the import map failed,
a SyntaxError or a TypeError will be thrown when parsing the import map text.

- The import-maps script tag needs to be placed before any module load.
This is one of the most common problems when using Import-maps. The import map
tag needs to be parsed before any module load happens, that includes:
  - Inline/external module load.
  - static import/dynamic import of Javascript modules.
  - Preload the module script in <modulepreload>.

- Resolution predence
See the 'Resolution precendence' part above, check if there is other specifier key
which takes a higher predence of the specifier key you thought.

## Why do we intentionally delay the implementation?

## Specification link
The specification could be found in [import-maps].

[import objects from other Javascript modules]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#importing_features_into_your_script
[Module specifier]: https://tc39.es/ecma262/#prod-ModuleSpecifier
[Module Specifier Map]: https://html.spec.whatwg.org/multipage/webappapis.html#module-specifier-map
[resolve a module specifier]: https://html.spec.whatwg.org/multipage/webappapis.html#resolve-a-module-specifier
[issue 235]: https://github.com/WICG/import-maps/issues/235
[issue 2]: https://github.com/WICG/import-maps/issues/2
[import-maps]: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps


---
[^1]: Currently external import maps are not supported, so you could only specify the import map in a HTML document.
[^2]: If it isn't a valid JSON string, a **SyntaxError** will be thrown. Otherwise, if the parsed strings are not of type JSON objects, a **TypeError** will be thrown.
