# The Parser Model

The parser model here follows the model in section
[8.2.1](http://www.w3.org/TR/2012/CR-html5-20121217/syntax.html#parsing)
of the HTML5 specification, though we do not assume a networking layer.

     [ InputStream ]    // Generic support for reading input.
           ||
      [ Scanner ]       // Breaks down the stream into characters.
           ||
     [ Tokenizer ]      // Groups characters into syntactic
           ||
    [ Tree Builder ]    // Organizes units into a tree of objects
           ||
     [ DOM Document ]     // The final state of the parsed document.


## InputStream

This is an interface with at least two concrete implementations:

- StringInputStream: Reads an HTML5 string.
- FileInputStream: Reads an HTML5 file.

## Scanner

This is a mechanical piece of the parser.

## Tokenizer

This follows section 8.4 of the HTML5 spec. It is (roughly) a recursive
descent parser. (Though there are plenty of optimizations that are less
than purely functional.

## EventHandler and DOMTree

EventHandler is the interface for tree builders. Since not all
implementations will necessarily build trees, we've chosen a more
generic name.

The event handler emits tokens during tokenization.

The DOMTree is an event handler that builds a DOM tree. The output of
the DOMTree builder is a DOMDocument.

### Using the event-based parser

For streaming-style processing where you do not need to build a DOM tree,
pass a custom `EventHandler` to `Tokenizer`. The tokenizer calls the handler
as it encounters start tags, end tags, text, comments, and the other parser
events.

```php
use Masterminds\HTML5\Elements;
use Masterminds\HTML5\Parser\EventHandler;
use Masterminds\HTML5\Parser\Scanner;
use Masterminds\HTML5\Parser\Tokenizer;

$events = new class implements EventHandler {
    public function doctype($name, $idType = 0, $id = null, $quirks = false)
    {
        printf("doctype: %s\n", $name);
    }

    public function startTag($name, $attributes = array(), $selfClosing = false)
    {
        printf("start: %s\n", $name);

        if (Elements::isA($name, Elements::TEXT_RAW)) {
            return Elements::TEXT_RAW;
        }

        if (Elements::isA($name, Elements::TEXT_RCDATA)) {
            return Elements::TEXT_RCDATA;
        }

        return 0;
    }

    public function endTag($name)
    {
        printf("end: %s\n", $name);
    }

    public function comment($cdata)
    {
        printf("comment: %s\n", $cdata);
    }

    public function text($cdata)
    {
        if ('' !== trim($cdata)) {
            printf("text: %s\n", trim($cdata));
        }
    }

    public function eof()
    {
        echo "eof\n";
    }

    public function parseError($msg, $line, $col)
    {
        fprintf(STDERR, "parse error at %d:%d: %s\n", $line, $col, $msg);
    }

    public function cdata($data)
    {
        printf("cdata: %s\n", $data);
    }

    public function processingInstruction($name, $data = null)
    {
        printf("processing instruction: %s %s\n", $name, (string) $data);
    }
};

$scanner = new Scanner('<p>Hello <strong>world</strong></p>');
$tokenizer = new Tokenizer($scanner, $events);
$tokenizer->parse();
```

`startTag()` may return one of the `Elements::TEXT_*` modes. This is how an
event handler tells the tokenizer to switch into raw-text or RCDATA parsing
for elements such as `script`, `style`, `textarea`, and `title`.

## DOMDocument

PHP has a DOMDocument class built-in (technically, it's part of libxml.)
We use that, thus rendering the output of this process compatible with
SimpleXML, QueryPath, and many other XML/HTML processing tools.

For cases where the HTML5 is a fragment of a HTML5 document a
DOMDocumentFragment is returned instead. This is another built-in class.
