# Whiskers - Mustache templates with extensions

A Dart library for parsing and rendering [Mustache templates](https://mustache.github.io/). It is derived from [mustache_template](https://pub.dev/packages/mustache_template) and adds an optional `onMissingVariable` callback to `renderString` and `render`. This makes it possible to extend templates with custom behavior. It is used by the web server in [Serverpod](https://serverpod.dev) for cache busting.

![Whiskers logo](https://raw.githubusercontent.com/serverpod/whiskers/main/misc/whiskers.webp)

See the [mustache manual](https://mustache.github.io/mustache.5.html) for detailed usage information.

This library passes all [mustache specification](https://github.com/mustache/spec/tree/master/specs) tests.

## Example usage
```dart
import 'package:whiskers/whiskers.dart';

main() {
	var source = '''
	  {{# names }}
            <div>{{ lastname }}, {{ firstname }}</div>
	  {{/ names }}
	  {{^ names }}
	    <div>No names.</div>
	  {{/ names }}
	  {{! I am a comment. }}
	''';

	var template = Template(source, name: 'template-filename.html');

	var output = template.renderString({'names': [
		{'firstname': 'Greg', 'lastname': 'Lowe'},
		{'firstname': 'Bob', 'lastname': 'Johnson'}
	]});

	print(output);
}
```

A template is parsed when it is created. After parsing, it can be rendered any
number of times with different values. A `TemplateException` is thrown if there
is a problem parsing or rendering the template.

The `Template` constructor allows passing a `name`. This name is used in error
messages. When working with multiple templates, it is helpful to pass a name so
that error messages clearly identify which template caused the error.

By default, all output from `{{variable}}` tags is HTML-escaped. This behavior
can be changed by passing `htmlEscapeValues: false` to the `Template`
constructor. You can also use a `{{{triple mustache}}}` tag or an unescaped
variable tag like `{{&unescaped}}`; output from those tags is not escaped.

## Handling missing variables

```dart
import 'package:whiskers/whiskers.dart';

void main() {
  final template = Template(
    '<script src="app.js?v={{cacheBuster}}"></script>',
    name: 'index.html',
  );

  final output = template.renderString(
    const {},
    onMissingVariable: (name, _) {
      if (name == 'cacheBuster') {
        return '20260408';
      }
      return null;
    },
  );

  print(output);
}
```

If `onMissingVariable` returns a string, that value is rendered in place of the
missing variable. If it returns `null`, the default behavior is preserved:
strict mode throws and lenient mode renders nothing.

## Differences between strict mode and lenient mode

### Strict mode (default)

* Tag names may only contain the characters a-z, A-Z, 0-9, underscore, period and minus. Other characters in tags will cause a TemplateException to be thrown during parsing.

* During rendering, if no map key or object member which matches the tag name is found, then a TemplateException will be thrown.

### Lenient mode

* Tag names may use any characters.
* During rendering, if no map key or object member which matches the tag name is found, then silently ignore and output nothing.

## Nested paths

```dart
  var t = Template('{{ author.name }}');
  var output = template.renderString({'author': {'name': 'Greg Lowe'}});
```

## Partials - example usage

```dart

var partial = Template('{{ foo }}', name: 'partial');

var resolver = (String name) {
   if (name == 'partial-name') { // Name of partial tag.
     return partial;
   }
};

var t = Template('{{> partial-name }}', partialResolver: resolver);

var output = t.renderString({'foo': 'bar'}); // bar

```

## Lambdas - example usage

```dart
var t = Template('{{# foo }}');
var lambda = (_) => 'bar';
t.renderString({'foo': lambda}); // bar
```

```dart
var t = Template('{{# foo }}hidden{{/ foo }}');
var lambda = (_) => 'shown';
t.renderString('foo': lambda); // shown
```

```dart
var t = Template('{{# foo }}oi{{/ foo }}');
var lambda = (LambdaContext ctx) => '<b>${ctx.renderString().toUpperCase()}</b>';
t.renderString({'foo': lambda}); // <b>OI</b>
```

```dart
var t = Template('{{# foo }}{{bar}}{{/ foo }}');
var lambda = (LambdaContext ctx) => '<b>${ctx.renderString().toUpperCase()}</b>';
t.renderString({'foo': lambda, 'bar': 'pub'}); // <b>PUB</b>
```

```dart
var t = Template('{{# foo }}{{bar}}{{/ foo }}');
var lambda = (LambdaContext ctx) => '<b>${ctx.renderString().toUpperCase()}</b>';
t.renderString({'foo': lambda, 'bar': 'pub'}); // <b>PUB</b>
```

In the following example `LambdaContext.renderSource(source)` re-parses the source string in the current context, this is the default behaviour in many mustache implementations. Since re-parsing the content is slow, and often not required, this library makes this step optional.

```dart
var t = Template('{{# foo }}{{bar}}{{/ foo }}');
var lambda = (LambdaContext ctx) => ctx.renderSource(ctx.source + ' {{cmd}}');
t.renderString({'foo': lambda, 'bar': 'pub', 'cmd': 'build'}); // pub build
```
