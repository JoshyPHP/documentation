## How it works

Almost every action performed by the s9e\TextFormatter library can be categorized as one of the following, mapped to their respective service:

 * Configuration → `text_formatter.s9e.factory`
 * Parsing → `text_formatter.s9e.parser` (aliased to `text_formatter.parser`)
 * Rendering → `text_formatter.s9e.renderer (aliased to `text_formatter.renderer)
 * Unparsing → `text_formatter.s9e.utils` (aliased to `text_formatter.utils`)

The actual services live in the `s9e` namespace but they are accessed through their non-namespaced aliases. The `text_formatter.s9e.factory` service doesn't have an alias because it is too specific to the library to have a generic interface.

Most of the configuration is done through the `s9e\TextFormatter\Configurator` object. Once configured, its `finalize()` method is called, which returns a `s9e\TextFormatter\Parser` object and a `s9e\TextFormatter\Renderer` object. Those objects can then be serialized and cached for future use. In phpBB, all of that happens in the `text_formatter.s9e.factory` service.

## How the data is transformed

Starting with the original, plain text:

```
[b]Hello world![/b] :)
```

The parser service will transform this text to XML:

```
<r><B><s>[b]</s>Hello world!<e>[/b]</e></B> <E>:)</E></r>
```

The XML is what is stored in the database. It can be transformed to HTML via the renderer service, or losslessly transformed back to plain text via the utils service.

```php
$original = '[b]Hello world![/b] :)';
$parsed   = $container->get('text_formatter.parser')->parse($text);
$rendered = $container->get('text_formatter.renderer')->render($parsed);
$unparsed = $container->get('text_formatter.utils')->unparsed($parsed);
```

## How it's integrated

Text formatting in 3.1 and earlier is performed via the `generate_text_for_*` global functions, as well as direct calls to `message_parser` methods or `decode_message()`.

Parsing is performed via `generate_text_for_storage()` and `message_parser::parse()`.
Rendering is performed via `generate_text_for_display()`.
Unparsing is performed via `generate_text_for_edit()` and `decode_message()`.

In 3.2, the text_formatter services are integrated directly into those functions to preserve backward compatibility. Functions that are expected to accept parsed text from 3.1 differentiate their input by examining its first 3 bytes. A text parsed by s9e\TextFormatter is stored in XML and will always start with a `r` or `t` element. Therefore, XML produced by s9e\TextFormatter always matches the following regular expression: `'#^<[rt][ >]#'`.