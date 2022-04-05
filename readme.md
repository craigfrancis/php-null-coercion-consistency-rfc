# PHP RFC: NULL Coercion Consistency

* Version: 0.1
* Voting Start: ?
* Voting End: ?
* RFC Started: 2022-04-05
* RFC Updated: 2022-04-05
* Author: Craig Francis [craig#at#craigfrancis.co.uk]
* Status: Draft
* First Published at: https://wiki.php.net/rfc/null_coercion_consistency
* GitHub Repo: https://github.com/craigfrancis/php-null-coercion-consistency-rfc
* Implementation: ?

## Introduction

PHP 8.1 introduced [Deprecate passing null to non-nullable arguments of internal functions](https://wiki.php.net/rfc/deprecate_null_to_scalar_internal_arg). While the consistency introduced by the RFC is welcome (user-defined vs internal functions), for those not using `strict_types=1`, the partial breaking of NULL coercion creates an upgrade problem.

This RFC does **not** change anything for scripts using `strict_types=1`, as type checks are useful in that context. For example, developers who view NULL as a missing/invalid value (not as a value in itself), consider passing NULL to a function like `htmlspecialchars()` as something that indicates a problem for them.

Roughly **85%** scripts do not use `strict_types=1` - This was calculated using [grep.app](https://grep.app/), to "search across a half million git repos", were each result is a script (not a count of matches,  [example](https://grep.app/search?q=defuse/php-encryption&filter[lang][0]=PHP)). We can see [269,701](https://grep.app/search?q=strict_types&filter[lang][0]=PHP) scripts using `strict_types=1`, out of [1,830,411](https://grep.app/search?q=php&filter[lang][0]=PHP). And keep in mind that [WordPress only really appears once](https://grep.app/search?q=class%20wpdb%20%7B&filter[lang][0]=PHP), it is [affected by this deprecation](https://make.wordpress.org/core/2022/01/10/wordpress-5-9-and-php-8-0-8-1/#php-8-1-deprecation-passing-null-to-non-nullable-php-native-functions-parameters), and is installed/used by many.

There was a [short discussion](https://externals.io/message/112327) about the original RFC, but with the exception of Craig Duncan, there was no consideration for the problems this creates with existing code (or the inconsistency of NULL coercion compared to string/int/float/bool coercion).

## Documentation

According to the documentation, when **not** using `strict_types=1`, "PHP will coerce values of the wrong type into the expected scalar type declaration if possible" ([ref](https://www.php.net/manual/en/language.types.declarations.php#language.types.declarations.strict)).

Where coercion from NULL is well defined:

[To String](https://www.php.net/manual/en/language.types.string.php): "null is always converted to an empty string."

[To Integer](https://www.php.net/manual/en/language.types.integer.php): "null is always converted to zero (0)."

[To Float](https://www.php.net/manual/en/language.types.float.php): "For values of other types, the conversion is performed by converting the value to int first and then to float"

[To Boolean](https://www.php.net/manual/en/language.types.boolean.php): "When converting to bool, the following values are considered false [...] the special type NULL"

For example:

```php
print('A');
print(1);
print(1.2);
print(false);
print(NULL); // Fine, coerced to empty string.

$o = [];

$o[] = ('' == '');
$o[] = ('' == NULL); // Fine, coerced to empty string.

$o[] = 'ConCat ' . 'A';
$o[] = 'ConCat ' . 123;
$o[] = 'ConCat ' . 1.2;
$o[] = 'ConCat ' . false;
$o[] = 'ConCat ' . NULL; // Fine, coerced to empty string.

$o[] = sprintf('%s', 'A');
$o[] = sprintf('%s', 1);
$o[] = sprintf('%s', 1.2);
$o[] = sprintf('%s', false);
$o[] = sprintf('%s', NULL); // Fine, coerced to empty string.

$o[] = htmlspecialchars('A');
$o[] = htmlspecialchars(1);
$o[] = htmlspecialchars(1.2);
$o[] = htmlspecialchars(false);
$o[] = htmlspecialchars(NULL); // Deprecated in 8.1, Fatal Error in 9.0?
```

Arrays, Resources, and Objects (without `__toString()`) cannot be coerced (for fairly obvious reasons).

String/Int/Float/Bool can be coerced.

NULL can usually be coerced, but...

- PHP 7.0 introduced the ability for user-defined functions to specify parameter types via the [Scalar Type Declarations RFC](https://wiki.php.net/rfc/scalar_type_hints_v5#behaviour_of_weak_type_checks), where the focus was on `strict_types=1`. But the implementation caused a Type Error when coercing NULL for everyone (even when not using `strict_types=1`), this seems more of an over-sight, with developers not using `strict_types=1` being unlikely to notice (as they won't specify types in their user defined functions).
- PHP 8.1 continued this inconsistency with internal function.

## Examples

Common sources of NULL:

```php
$search = (isset($_GET['q']) ? $_GET['q'] : NULL);

$search = ($_GET['q'] ?? NULL); // Since PHP 7

$search = filter_input(INPUT_GET, 'q');

$search = $request->input('q'); // Laravel
$search = $request->get('q'); // Symfony
$search = $this->request->getQuery('q'); // CakePHP
$search = $request->getGet('q'); // CodeIgniter

$value = array_pop($empty_array);
$value = mysqli_fetch_row($result);
$value = json_decode($json); // Invalid JSON, or deeper than the nesting limit.
```

Examples where NULL has been fine for scripts not using `strict_types=1`:

```php
$search_trimmed = trim($search);

$search_len = strlen($search);

$search_upper = strtoupper($search);

$search_hash = hash('sha256', $search);

echo htmlspecialchars($search);

echo 'https://example.com/?q=' . urlencode($search);

preg_match('/^[a-z]/', $search);

exec('/path/to/cmd ' . escapeshellarg($search));

socket_write($socket, $search);

xmlwriter_text($writer, $search);
```

And developers have used `NULL` to skip certain parameters, e.g.

```php
setcookie('q', $search, NULL, NULL, NULL, true, true); // x4

mail('nobody@example.com', 'subject', 'message', NULL, '-fwebmaster@example.com');
```

There are approximately [335 parameters affected by this deprecation](https://github.com/craigfrancis/php-allow-null-rfc/blob/main/functions-change.md).

As an aside, there are also roughly [104 questionable](https://github.com/craigfrancis/php-allow-null-rfc/blob/main/functions-maybe.md) and [558 problematic](https://github.com/craigfrancis/php-allow-null-rfc/blob/main/functions-other.md) parameters which probably should not accept NULL **or** an Empty String. For example, `$separator` in `explode()` already has a "cannot be empty" Fatal Error. For these parameters, a different RFC could consider updating them to reject both NULL and Empty Strings, e.g. $needle in strpos(), and $json in json_decode().

## Finding

The only realistic way for developers to find these issues is to use the deprecation notices (not ideal).

And while it's possible to use very strict Static Analysis, which follows every variable from source to sink (to check if a variable can be `NULL`), most developers are not in a position to do this.

In the last JetBrains developer survey, with 67% of the responses regularly using Laravel, **only 33% used Static Analysis** ([source](https://www.jetbrains.com/lp/devecosystem-2021/php/#PHP_do-you-use-static-analysis)).

As an example, take this simple script:

```php
./src/index.php
<?php
$nullable = ($_GET['a'] ?? NULL);
echo htmlentities($nullable);
?>
```

It's considered fine today by the relevant tools:

```cli
composer require --dev vimeo/psalm
./vendor/bin/psalm --init ./src/ 4
./vendor/bin/psalm
No errors found!
```
Note: Psalm can detect this problem at [levels 1, 2, and 3](https://psalm.dev/docs/running_psalm/error_levels/) (don't use a baseline).

```cli
composer require --dev phpstan/phpstan
./vendor/bin/phpstan analyse -l 9 ./src/
[OK] No errors
```

```cli
composer require --dev phpstan/phpstan-strict-rules
composer require --dev phpstan/extension-installer
./vendor/bin/phpstan analyse -l 9 ./src/
[OK] No errors
```
Note: There are [Stricter Analysis](https://phpstan.org/config-reference#stricter-analysis) options for PHPStan, but they don't seem to help with this problem.

```cli
composer require --dev rector/rector
./vendor/bin/rector init
./vendor/bin/rector process ./src/
[OK] Rector is done!
```

```cli
composer require --dev "squizlabs/php_codesniffer=*"
./vendor/bin/phpcs -p ./src/
E 1 / 1 (100%)
[...]
 2 | ERROR | Missing file doc comment
[...]
```

```cli
composer require friendsofphp/php-cs-fixer
./vendor/bin/php-cs-fixer fix src --diff --allow-risky=yes
Loaded config default.
Using cache file ".php-cs-fixer.cache".
   1) src/index.php
      ---------- begin diff ----------
--- src/index.php
+++ src/index.php
@@ -1,4 +1,4 @@
 <?php
+
 $nullable = ($_GET['a'] ?? null);
 echo htmlentities($nullable);
-?>
\ No newline at end of file

      ----------- end diff -----------

Fixed all files in 0.012 seconds, 12.000 MB memory used
```

```cli
composer require --dev phpcompatibility/php-compatibility
sed -i '' -E 's/(PHPCSHelper::getConfigData)/(string) \1/g' vendor/phpcompatibility/php-compatibility/PHPCompatibility/Sniff.php
./vendor/bin/phpcs --config-set installed_paths vendor/phpcompatibility/php-compatibility
./vendor/bin/phpcs -p ./src/ --standard=PHPCompatibility --runtime-set testVersion 8.1
. 1 / 1 (100%)
```

Note: Juliette (@jrfnl) has confirmed that getting PHPCompatibility to solve this problem will be "pretty darn hard to do" because it's "not reliably sniffable" ([source](https://twitter.com/jrf_nl/status/1497937320766496772)).

## Temporary Solutions

You can disable `E_DEPRECATED` (as recommended by projects like WordPress).

Alternatively you can use `set_error_handler()`, with something like:

```php
function ignore_null_coercion($errno, $errstr) {
  // https://github.com/php/php-src/blob/012ef7912a8a0bb7d11b2dc8d108cc859c51e8d7/Zend/zend_API.c#L458
  if ($errno === E_DEPRECATED && preg_match('/Passing null to parameter #.* of type .* is deprecated/', $errstr)) {
    return true;
  }
  return false;
}
set_error_handler('ignore_null_coercion', E_DEPRECATED);
```

And some developers are simply [patching php-src](https://externals.io/message/116519#116559) (risky).

## Fixing

While individual changes are easy, they are very difficult to find, there are many of them (time consuming), and the updates used are often pointless, e.g.

* `urlencode(strval($name));`
* `urlencode((string) $name);`
* `urlencode($name ?? "");`

One example diff didn't exactly make the code easier to read:

```diff
 - $result = substr($string, 0, $length);
 + $result = substr($string ?? '', 0, $length);
```

As noted above - PHPCompatibility, CodeSniffer, Rector, etc are unable to fix this issue.

## Proposal

No change for those using `strict_types=1`.

Must keep the spirit of the original RFC, and keep user-defined and internal functions consistent.

But continue to support NULL coercion into an empty string, the integer/float 0, or the boolean false.

The current intention of causing Fatal Errors in 9.0 will make upgrading very difficult.

## Backward Incompatible Changes

None

## Proposed PHP Version(s)

PHP 8.2

## RFC Impact

### To SAPIs

None known

### To Existing Extensions

None known

### To Opcache

None known

## Open Issues

"terrible idea" - I'm still waiting to hear details.

"it's a bit late" - We only have a deprecation at the moment (can be ignored), it will be "too late" when PHP 9.0 uses Fatal Errors.

## Future Scope

Some function parameters could be updated to complain when an `NULL` **or** an `Empty String` is provided; e.g. `$method` in `method_exists()`, or `$characters` in `trim()`.

## Voting

Accept the RFC

TODO

## Implementation

TODO

## Rejected Features

Updating some parameters to accept NULL ([details](https://wiki.php.net/rfc/allow_null)).

## Notes

In PHP 7.0, in the [Scalar Type Declarations](https://wiki.php.net/rfc/scalar_type_hints_v5#behaviour_of_weak_type_checks) RFC, scalar types were defined as "int, float, string and bool" - but, despite NULL also being a simple value (i.e. not an array/object/resource), it was not included in this definition.

Also, note the example quote from [Rasmus](http://news.php.net/php.internals/71525):

> PHP is and should remain:
> 1) a pragmatic web-focused language
> 2) a loosely typed language
> 3) a language which caters to the skill-levels and platforms of a wide range of users
