svg-sprite [![NPM version][npm-image]][npm-url] [![Build Status][travis-image]][travis-url]  [![Coverage Status][coveralls-image]][coveralls-url] [![Dependency Status][depstat-image]][depstat-url]
==========

is a low-level [Node.js](http://nodejs.org/) module that **takes a bunch of [SVG](http://www.w3.org/TR/SVG/) files**, optimizes them and bakes them into **SVG sprites** of several types:

*	Traditional **[CSS sprites](http://en.wikipedia.org/wiki/Sprite_(computer_graphics)#Sprites_by_CSS)** for use as background images,
*	CSS sprites with **pre-defined `<view>` elements**, useful for foreground images as well,
*	inline sprites using the **`<defs>` element**,
*	inline sprites using the **`<symbol>` element**
*	and **[SVG stacks](http://simurai.com/blog/2012/04/02/svg-stacks/)**.

It comes with a set of **[Mustache](http://mustache.github.io/) templates** for creating stylesheets in good ol' [CSS](http://www.w3.org/Style/CSS/) or one of the major **pre-processor formats** ([Sass](http://sass-lang.com/), [Less](http://lesscss.org/) and [Stylus](http://learnboost.github.io/stylus/)). Tweaking the templates or even adding your own **custom output format** is really easy, just as switching on the generation of an **HTML example document** along with your sprite.

Grunt, Gulp & Co.
-----------------

Being a low-level library with support for [Node.js streams](https://github.com/substack/stream-handbook), *svg-sprite* doesn't take on the part of accessing the file system (i.e. reading the source SVGs from and writing the sprites and CSS files to disk). If you don't want to take care of this stuff yourself, you might rather have a look at the available wrappers for **Grunt** ([grunt-svg-sprite](https:// github.com/jkphl/grunt-svg-sprite)) and **Gulp** ([gulp-svg-sprite](https://github.com/jkphl/gulp-svg-sprite)). *svg-sprite* is also the foundation of the **[iconizr](https://github.com/jkphl/node-iconizr)** project, which serves high-quality SVG based **CSS icon kits with PNG fallbacks**.

Table of contents
-----------------
* [Installation](#installation)
* [Getting started](#getting-started)
	* [Usage pattern](#usage-pattern)
* [Configuration basics](#configuration-basics)
	* [General configuration options](#general-configuration-options)
	* [Output modes](#output-modes)
		* [Common mode properties](#common-mode-properties)
		* [Basic examples](#basic-examples)
	* [Online configurator & project kickstarter](http://jkphl.github.io/svg-sprite)
* [Advanced techniques](#advanced-techniques)
	* [Templating](#templating)
* [Command line usage](#command-line-usage)
* [Known problems / To-do](#known-problems--to-do)
* [Changelog](CHANGELOG.md)
* [Legal](#legal)

Installation
------------

To install *svg-sprite* globally, run

```bash
npm install svg-sprite -g
```

on the command line.

Getting started
---------------

Crafting a sprite with *svg-sprite* typically follows these steps:

1. You [create an instance of the SVGSpriter](docs/api.md#svgspriter-config-), passing a main configuration object to the constructor.
2. You [register a couple of SVG source files](docs/api.md#svgspriteraddfile--name-svg-) for processing.
3. You [trigger the compilation process](docs/api.md#svgspritercompile-config--callback-) and receive the generated files (sprite, CSS, example documents etc.) .

The procedure is the very same for all supported sprite types («modes»).

### Usage pattern

```javascript
// Create spriter instance (see below for `config` examples)
var spriter       = new SVGSpriter(config);

// Add SVG source files — the manual way ...
spriter.add('assets/svg-1.svg', null, fs.readFileSync('assets/svg-1.svg', {encoding: 'utf-8'}));
spriter.add('assets/svg-2.svg', null, fs.readFileSync('assets/svg-2.svg', {encoding: 'utf-8'}));
	/* ... */

// Compile the sprite
spriter.compile(function(error, result) {
	/* ... Write `result` files to disk or do whatever with them ... */
});
```

As you see, big parts of the above deal with disk I/O related stuff. You can make your life easier by [using the Grunt or Gulp modules](docs/grunt-gulp.md#basic-usage-pattern) instead of the *svg-sprite* [default API](docs/api.md).

Configuration basics
--------------------

Did you notice the `config` variable that is passed to the constructor in the example above? This is *svg-sprite*'s **main configuration** — an `Object` with the following properties:

```javascript
{
	dest			: <String>,				// Main output directory
	log  			: <String∣Logger>,		// Logging verbosity or custom logger
	shape			: <Object>,				// SVG shape configuration
	transform		: <Array>,				// SVG transformations
	svg				: <Object>,				// Common SVG options
	variables		: <Object>,				// Custom templating variables
	mode			: <Object>				// Output mode configuration
}
```

If you don't provide a configuration altogether, *svg-sprite* uses built-in defaults for these properties, so in fact they are all optional. However, you will need to enable at least one **output mode** (`mode` property) to get some reasonable results (i.e. a sprite of some type).

### General configuration options

The common configuration properties (all except `mode`) apply to all sprites created by a single spriter instance. Their default values are:

```javascript
// Common svg-sprite config options and their defaul values

var config					= {
	dest					: '.',						// Main output directory
	log						: null,						// Logging verbosity (default: no logging)
	shape					: {							// SVG shape related options
		id					: {							// SVG shape ID related options
			separator		: '--',						// Separator for directory name traversal
			generator		: function() { /*...*/ },	// SVG shape ID generator callback
			pseudo			: '~'						// File name separator for shape states (e.g. ':hover')
		},
		dimension			: {							// Dimension related options
			maxWidth		: 2000,						// Max. shape width
			maxHeight		: 2000,						// Max. shape height
			precision		: 2,						// Floating point precision
			attributes		: false,					// Width and height attributes on embedded shapes
		},
		spacing				: {							// Spacing related options
			padding			: 0,						// Padding around all shapes
			box				: 'content'					// Padding strategy (similar to CSS `box-sizing`)
		},
		meta				: null,						// Path to YAML file with meta / accessibility data
		align				: null,						// Path to YAML file with extended alignment data
		dest				: null						// Output directory for optimized intermediate SVG shapes
	},
	transform				: ['svgo'],					// List of transformations / optimizations
	svg						: {							// General options for all SVG output
		xmlDeclaration		: true,						// Add XML declaration to SVG sprite
		doctypeDeclaration	: true,						// Add DOCTYPE declaration to SVG sprite
		namespaceIDs		: true,						// Add namespace token to all IDs in SVG shapes
		dimensionAttributes	: true						// Width and height attributes on the sprite
	},
	variables				: {}						// Custom Mustache templating variables and functions
}
```

Please refer to the [configuration documentation](docs/configuration.md) for details.

### Output modes

At the moment, *svg-sprite* supports **five different output modes** (i.e. sprite types), each of them having it's own characteristics and use cases. It's up to you to decide which sprite type is the best choice for your project. The `mode` option controls which sprite types are created. You may enable more than one output mode at a time — *svg-sprite* will happily create several sprites in parallel.

To enable the creation of a specific sprite type with default values, simply set the appropriate `mode` property to `true`:

```javascript
var config					= {
	mode					:
		css					: true,		// Create a «css» sprite
		view				: true,		// Create a «view» sprite
		defs				: true,		// Create a «defs» sprite
		symbol				: true,		// Create a «symbol» sprite
		stack				: true		// Create a «stack» sprite
	}
}
```

To further configure a sprite, pass in an object with configuration options:

```javascript
// «symbol» sprite with CSS stylesheet resource

var config					= {
	mode					:
		css					: {
			// Configuration for the «css» sprite
			// ...
		}
	}
}
```

#### Common mode properties

Most of the `mode` properties are shared between the sprite types:

```javascript
// Common mode properties

var config					= {
	mode					:
		<mode> 				: {
			dest			: "<mode>",						// Mode specific output directory
			prefix			: "svg-%s",						// Prefix for CSS selectors
			dimensions		: "-dims",						// Suffix for dimension CSS selectors
			sprite			: "svg/sprite.<mode>.svg"		// Sprite path and name
			bust			: true|false,					// Cache busting (default value is mode dependent)
			example			: false							// Create HTML example document
		}
	}
}
```

Depending on the sprite type there may be additional settings. Please refer to the [configuration documentation](docs/configuration.md) for a full reference.

#### Basic examples

##### A.) Standalone sprite

Foreground image **sprite with `<symbol>` elements** (for being `<use>`d in your HTML source):

```javascript
// «symbol» sprite with CSS stylesheet resource

var config					= {
	mode					:
		symbol				: true		// Create a «symbol» sprite
	}
}
```

##### B.) Sprite with CSS resource

Traditional **CSS sprite** along with a **plain CSS stylesheet**:

```javascript
// «css» sprite with CSS stylesheet resource

var config					= {
	mode					:
		css					: {			// Create a «css» sprite
			render			: {
				css			: true		// Render a CSS stylesheet
			}
		}
	}
}
```

##### C.) Multiple sprites

**`<defs>` sprite**, **`<symbol>` sprite** and an **SVG stack** all at once:

```javascript
// «defs», «symbol» and «stack» sprites in parallel

var config					= {
	mode					:
		defs				: true,
		symbol				: true,
		stack				: true
	}
}
```

##### D.) No sprite at all

`mode`-less run, returning the **optimized SVG shapes only**:

```javascript
// Just optimize source SVG files, create no sprite

var config					= {
	shape					:
		dest				: 'path/to/out/dir'
	}
}
```
> **NOTE ABOUT THE `dest` OPTION(S)**
> 
> "Didn't you say that *svg-sprite* doesn't access the file system? So why do you need an output directory?" — Well, good point. *svg-sprite* uses [vinyl](https://github.com/wearefractal/vinyl) file objects to pass along virtual resources and to specify where they **are intended to be located**. This is especially important for relative file contexts (e.g. the path to the SVG sprite from the perspective of a referencing CSS stylesheet).


### Online configurator & project kickstarter

To get you off the ground quickly, I made a simple [online configurator](http://jkphl.github.io/svg-sprite) that lets you create a custom *svg-sprite* configuration in seconds. You may download it as plain JSON, Node.js project, Gruntfile or Gulpfile. Please visit the configurator at http://jkphl.github.io/svg-sprite.


Advanced techniques
-------------------

### Templating



Known problems / To-do
----------------------

* SVGO does not minify element IDs when there are `<style>` or `<script>` elements contained in the file

Legal
-----
Copyright © 2015 Joschi Kuphal <joschi@kuphal.net> / [@jkphl](https://twitter.com/jkphl)

*svg-sprite* is licensed under the terms of the [MIT license](LICENSE.txt).

The contained example SVG icons are part of the [Tango Icon Library](http://tango.freedesktop.org/Tango_Icon_Library) and belong to the Public Domain.


[npm-url]: https://npmjs.org/package/svg-sprite
[npm-image]: https://badge.fury.io/js/svg-sprite.png

[travis-url]: http://travis-ci.org/jkphl/svg-sprite
[travis-image]: https://secure.travis-ci.org/jkphl/svg-sprite.png

[coveralls-url]: https://coveralls.io/r/jkphl/svg-sprite
[coveralls-image]: https://img.shields.io/coveralls/jkphl/svg-sprite.svg

[depstat-url]: https://david-dm.org/jkphl/svg-sprite
[depstat-image]: https://david-dm.org/jkphl/svg-sprite.svg