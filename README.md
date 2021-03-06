PHP CSS Parser
--------------

A Parser for CSS Files written in PHP. Allows extraction of CSS files into a data structure, manipulation of said structure and output as (optimized) CSS.

## Usage

### Installation using composer

Add php-css-parser to your composer.json

        {
            "require": {
                "sabberworm/php-css-parser": "*"
            }
        }

### Extraction

To use the CSS Parser, create a new instance. The constructor takes the following form:

	new Sabberworm\CSS\Parser($sText, $sDefaultCharset = 'utf-8');

The charset is used only if no @charset declaration is found in the CSS file.

To read a file, for example, you’d do the following:

	$oCssParser = new Sabberworm\CSS\Parser(file_get_contents('somefile.css'));
	$oCssDocument = $oCssParser->parse();

The resulting CSS document structure can be manipulated prior to being output.

### Manipulation

The resulting data structure consists mainly of five basic types: `CSSList`, `RuleSet`, `Rule`, `Selector` and `Value`. There are two additional types used: `Import` and `Charset` which you won’t use often.

#### CSSList

`CSSList` represents a generic CSS container, most likely containing declaration blocks (rule sets with a selector) but it may also contain at-rules, charset declarations, etc. `CSSList` has the following concrete subtypes:

* `Document` – representing the root of a CSS file.
* `MediaQuery` – represents a subsection of a CSSList that only applies to a output device matching the contained media query.

#### RuleSet

`RuleSet` is a container for individual rules. The most common form of a rule set is one constrained by a selector. The following concrete subtypes exist:

* `AtRule` – for generic at-rules which do not match the ones specifically mentioned like @import, @charset or @media. A common example for this is @font-face.
* `DeclarationBlock` – a RuleSet constrained by a `Selector`; contains an array of selector objects (comma-separated in the CSS) as well as the rules to be applied to the matching elements.

Note: A `CSSList` can contain other `CSSList`s (and `Import`s as well as a `Charset`) while a `RuleSet` can only contain `Rule`s.

#### Rule

`Rule`s just have a key (the rule) and a value. These values are all instances of a `Value`.

#### Value

`Value` is an abstract class that only defines the `__toString` method. The concrete subclasses for atomic value types are:

* `Size` – consists of a numeric `size` value and a unit.
* `Color` – colors can be input in the form #rrggbb, #rgb or schema(val1, val2, …) but are alwas stored as an array of ('s' => val1, 'c' => val2, 'h' => val3, …) and output in the second form.
* `String` – this is just a wrapper for quoted strings to distinguish them from keywords; always output with double quotes.
* `URL` – URLs in CSS; always output in URL("") notation.

There is another abstract subclass of `Value`, `ValueList`. A `ValueList` represents a lists of `Value`s, separated by some separation character (mostly `,`, whitespace, or `/`). There are two types of `ValueList`s:

* `RuleValueList` – The default type, used to represent all multi-valued rules like `font: bold 12px/3 Helvetica, Verdana, sans-serif;` (where the value would be a whitespace-separated list of the primitive value `bold`, a slash-separated list and a comma-separated list).
* `CSSFunction` – A special kind of value that also contains a function name and where the values are the function’s arguments. Also handles equals-sign-separated argument lists like `filter: alpha(opacity=90);`.

To access the items stored in a `CSSList` – like the document you got back when calling `$oCssParser->parse()` –, use `getContents()`, then iterate over that collection and use instanceof to check whether you’re dealing with another `CSSList`, a `RuleSet`, a `Import` or a `Charset`.

To append a new item (selector, media query, etc.) to an existing `CSSList`, construct it using the constructor for this class and use the `append($oItem)` method.

If you want to manipulate a `RuleSet`, use the methods `addRule(Rule $oRule)`, `getRules()` and `removeRule($mRule)` (which accepts either a Rule instance or a rule name; optionally suffixed by a dash to remove all related rules).

#### Convenience methods

There are a few convenience methods on Document to ease finding, manipulating and deleting rules:

* `getAllDeclarationBlocks()` – does what it says; no matter how deeply nested your selectors are. Aliased as `getAllSelectors()`.
* `getAllRuleSets()` – does what it says; no matter how deeply nested your rule sets are.
* `getAllValues()` – finds all `Value` objects inside `Rule`s.

### Use cases

#### Use `Parser` to prepend an id to all selectors

	$sMyId = "#my_id";
	$oParser = new Sabberworm\CSS\Parser($sText);
	$oCss = $oParser->parse();
	foreach($oCss->getAllDeclarationBlocks() as $oBlock) {
		foreach($oBlock->getSelectors() as $oSelector) {
			//Loop over all selector parts (the comma-separated strings in a selector) and prepend the id
			$oSelector->setSelector($sMyId.' '.$oSelector->getSelector());
		}
	}
	
#### Shrink all absolute sizes to half

	$oParser = new Sabberworm\CSS\Parser($sText);
	$oCss = $oParser->parse();
	foreach($oCss->getAllValues() as $mValue) {
		if($mValue instanceof CSSSize && !$mValue->isRelative()) {
			$mValue->setSize($mValue->getSize()/2);
		}
	}

#### Remove unwanted rules

	$oParser = new Sabberworm\CSS\Parser($sText);
	$oCss = $oParser->parse();
	foreach($oCss->getAllRuleSets() as $oRuleSet) {
		$oRuleSet->removeRule('font-'); //Note that the added dash will make this remove all rules starting with font- (like font-size, font-weight, etc.) as well as a potential font-rule
		$oRuleSet->removeRule('cursor');
	}

### Output

To output the entire CSS document into a variable, just use `->__toString()`:

	$oCssParser = new Sabberworm\CSS\Parser(file_get_contents('somefile.css'));
	$oCssDocument = $oCssParser->parse();
	print $oCssDocument->__toString();

## Examples

### Example 1 (At-Rules)

#### Input

	@charset "utf-8";

	@font-face {
	  font-family: "CrassRoots";
	  src: url("../media/cr.ttf")
	}
	
	html, body {
		font-size: 1.6em
	}
	
#### Structure (`var_dump()`)

	object(Sabberworm\CSS\CSSList\Document)#219 (1) {
	  ["aContents":"Sabberworm\CSS\CSSList\CSSList":private]=>
	  array(3) {
	    [0]=>
	    object(Sabberworm\CSS\Property\Charset)#11 (1) {
	      ["sCharset":"Sabberworm\CSS\Property\Charset":private]=>
	      object(Sabberworm\CSS\Value\String)#220 (1) {
	        ["sString":"Sabberworm\CSS\Value\String":private]=>
	        string(5) "utf-8"
	      }
	    }
	    [1]=>
	    object(Sabberworm\CSS\RuleSet\AtRule)#6 (2) {
	      ["sType":"Sabberworm\CSS\RuleSet\AtRule":private]=>
	      string(9) "font-face"
	      ["aRules":"Sabberworm\CSS\RuleSet\RuleSet":private]=>
	      array(2) {
	        ["font-family"]=>
	        object(Sabberworm\CSS\Rule\Rule)#15 (3) {
	          ["sRule":"Sabberworm\CSS\Rule\Rule":private]=>
	          string(11) "font-family"
	          ["mValue":"Sabberworm\CSS\Rule\Rule":private]=>
	          object(Sabberworm\CSS\Value\String)#13 (1) {
	            ["sString":"Sabberworm\CSS\Value\String":private]=>
	            string(10) "CrassRoots"
	          }
	          ["bIsImportant":"Sabberworm\CSS\Rule\Rule":private]=>
	          bool(false)
	        }
	        ["src"]=>
	        object(Sabberworm\CSS\Rule\Rule)#12 (3) {
	          ["sRule":"Sabberworm\CSS\Rule\Rule":private]=>
	          string(3) "src"
	          ["mValue":"Sabberworm\CSS\Rule\Rule":private]=>
	          object(Sabberworm\CSS\Value\URL)#14 (1) {
	            ["oURL":"Sabberworm\CSS\Value\URL":private]=>
	            object(Sabberworm\CSS\Value\String)#16 (1) {
	              ["sString":"Sabberworm\CSS\Value\String":private]=>
	              string(15) "../media/cr.ttf"
	            }
	          }
	          ["bIsImportant":"Sabberworm\CSS\Rule\Rule":private]=>
	          bool(false)
	        }
	      }
	    }
	    [2]=>
	    object(Sabberworm\CSS\RuleSet\DeclarationBlock)#17 (2) {
	      ["aSelectors":"Sabberworm\CSS\RuleSet\DeclarationBlock":private]=>
	      array(2) {
	        [0]=>
	        object(Sabberworm\CSS\Property\Selector)#18 (2) {
	          ["sSelector":"Sabberworm\CSS\Property\Selector":private]=>
	          string(4) "html"
	          ["iSpecificity":"Sabberworm\CSS\Property\Selector":private]=>
	          NULL
	        }
	        [1]=>
	        object(Sabberworm\CSS\Property\Selector)#19 (2) {
	          ["sSelector":"Sabberworm\CSS\Property\Selector":private]=>
	          string(4) "body"
	          ["iSpecificity":"Sabberworm\CSS\Property\Selector":private]=>
	          NULL
	        }
	      }
	      ["aRules":"Sabberworm\CSS\RuleSet\RuleSet":private]=>
	      array(1) {
	        ["font-size"]=>
	        object(Sabberworm\CSS\Rule\Rule)#20 (3) {
	          ["sRule":"Sabberworm\CSS\Rule\Rule":private]=>
	          string(9) "font-size"
	          ["mValue":"Sabberworm\CSS\Rule\Rule":private]=>
	          object(Sabberworm\CSS\Value\Size)#21 (3) {
	            ["fSize":"Sabberworm\CSS\Value\Size":private]=>
	            float(1.6)
	            ["sUnit":"Sabberworm\CSS\Value\Size":private]=>
	            string(2) "em"
	            ["bIsColorComponent":"Sabberworm\CSS\Value\Size":private]=>
	            bool(false)
	          }
	          ["bIsImportant":"Sabberworm\CSS\Rule\Rule":private]=>
	          bool(false)
	        }
	      }
	    }
	  }
	}

#### Output (`__toString()`)

	@charset "utf-8";@font-face {font-family: "CrassRoots";src: url("../media/cr.ttf");}html, body {font-size: 1.6em;}

### Example 2 (Values)

#### Input

	#header {
		margin: 10px 2em 1cm 2%;
		font-family: Verdana, Helvetica, "Gill Sans", sans-serif;
		color: red !important;
	}
	
#### Structure (`var_dump()`)

	object(Sabberworm\CSS\CSSList\Document)#217 (1) {
	  ["aContents":"Sabberworm\CSS\CSSList\CSSList":private]=>
	  array(1) {
	    [0]=>
	    object(Sabberworm\CSS\RuleSet\DeclarationBlock)#17 (2) {
	      ["aSelectors":"Sabberworm\CSS\RuleSet\DeclarationBlock":private]=>
	      array(1) {
	        [0]=>
	        object(Sabberworm\CSS\Property\Selector)#20 (2) {
	          ["sSelector":"Sabberworm\CSS\Property\Selector":private]=>
	          string(7) "#header"
	          ["iSpecificity":"Sabberworm\CSS\Property\Selector":private]=>
	          NULL
	        }
	      }
	      ["aRules":"Sabberworm\CSS\RuleSet\RuleSet":private]=>
	      array(3) {
	        ["margin"]=>
	        object(Sabberworm\CSS\Rule\Rule)#21 (3) {
	          ["sRule":"Sabberworm\CSS\Rule\Rule":private]=>
	          string(6) "margin"
	          ["mValue":"Sabberworm\CSS\Rule\Rule":private]=>
	          object(Sabberworm\CSS\Value\RuleValueList)#14 (2) {
	            ["aComponents":protected]=>
	            array(4) {
	              [0]=>
	              object(Sabberworm\CSS\Value\Size)#19 (3) {
	                ["fSize":"Sabberworm\CSS\Value\Size":private]=>
	                float(10)
	                ["sUnit":"Sabberworm\CSS\Value\Size":private]=>
	                string(2) "px"
	                ["bIsColorComponent":"Sabberworm\CSS\Value\Size":private]=>
	                bool(false)
	              }
	              [1]=>
	              object(Sabberworm\CSS\Value\Size)#18 (3) {
	                ["fSize":"Sabberworm\CSS\Value\Size":private]=>
	                float(2)
	                ["sUnit":"Sabberworm\CSS\Value\Size":private]=>
	                string(2) "em"
	                ["bIsColorComponent":"Sabberworm\CSS\Value\Size":private]=>
	                bool(false)
	              }
	              [2]=>
	              object(Sabberworm\CSS\Value\Size)#6 (3) {
	                ["fSize":"Sabberworm\CSS\Value\Size":private]=>
	                float(1)
	                ["sUnit":"Sabberworm\CSS\Value\Size":private]=>
	                string(2) "cm"
	                ["bIsColorComponent":"Sabberworm\CSS\Value\Size":private]=>
	                bool(false)
	              }
	              [3]=>
	              object(Sabberworm\CSS\Value\Size)#12 (3) {
	                ["fSize":"Sabberworm\CSS\Value\Size":private]=>
	                float(2)
	                ["sUnit":"Sabberworm\CSS\Value\Size":private]=>
	                string(1) "%"
	                ["bIsColorComponent":"Sabberworm\CSS\Value\Size":private]=>
	                bool(false)
	              }
	            }
	            ["sSeparator":protected]=>
	            string(1) " "
	          }
	          ["bIsImportant":"Sabberworm\CSS\Rule\Rule":private]=>
	          bool(false)
	        }
	        ["font-family"]=>
	        object(Sabberworm\CSS\Rule\Rule)#16 (3) {
	          ["sRule":"Sabberworm\CSS\Rule\Rule":private]=>
	          string(11) "font-family"
	          ["mValue":"Sabberworm\CSS\Rule\Rule":private]=>
	          object(Sabberworm\CSS\Value\RuleValueList)#13 (2) {
	            ["aComponents":protected]=>
	            array(4) {
	              [0]=>
	              string(7) "Verdana"
	              [1]=>
	              string(9) "Helvetica"
	              [2]=>
	              object(Sabberworm\CSS\Value\String)#15 (1) {
	                ["sString":"Sabberworm\CSS\Value\String":private]=>
	                string(9) "Gill Sans"
	              }
	              [3]=>
	              string(10) "sans-serif"
	            }
	            ["sSeparator":protected]=>
	            string(1) ","
	          }
	          ["bIsImportant":"Sabberworm\CSS\Rule\Rule":private]=>
	          bool(false)
	        }
	        ["color"]=>
	        object(Sabberworm\CSS\Rule\Rule)#11 (3) {
	          ["sRule":"Sabberworm\CSS\Rule\Rule":private]=>
	          string(5) "color"
	          ["mValue":"Sabberworm\CSS\Rule\Rule":private]=>
	          string(3) "red"
	          ["bIsImportant":"Sabberworm\CSS\Rule\Rule":private]=>
	          bool(true)
	        }
	      }
	    }
	  }
	}

#### Output (`__toString()`)

	#header {margin: 10px 2em 1cm 2%;font-family: Verdana,Helvetica,"Gill Sans", sans-serif;color: red !important;}

## To-Do

* More convenience methods [like `selectorsWithElement($sId/Class/TagName)`, `removeSelector($oSelector)`, `attributesOfType($sType)`, `removeAttributesOfType($sType)`]
* Options for output (compact, verbose, etc.)
* Named color support (using `Color` instead of an anonymous string literal)
* Test suite
* Adopt lenient parsing rules
* Support for @-rules that are CSSLists (other than @media and @keyframes).

## Contributors/Thanks to

* [ju1ius](https://github.com/ju1ius) for the specificity parsing code and the ability to expand/compact shorthand properties.
* [GaryJones](https://github.com/GaryJones) for lots of input and [http://css-specificity.info/](http://css-specificity.info/).
* [docteurklein](https://github.com/docteurklein) for output formatting and `CSSList->remove()` inspiration.
* [nicolopignatelli](https://github.com/nicolopignatelli) for PSR-0 compatibility.
* [diegoembarcadero](https://github.com/diegoembarcadero) for keyframe at-rule parsing.
* [goetas](https://github.com/goetas) for @namespace at-rule support.

## Misc

* Legacy Support: The latest pre-PSR-0 version of this project can be checked out as the `v0.9` tag.
* Running Tests: To run all unit tests for this project, have `phpunit` installed, `cd` to the `tests` dir and run `phpunit --bootstrap bootstrap.php .`.

## License

PHP-CSS-Parser is freely distributable under the terms of an MIT-style license.

Copyright (c) 2011 Raphael Schweikert, http://sabberworm.com/

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
