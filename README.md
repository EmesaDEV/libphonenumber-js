# Emesa custom build

Reduce size of the library including only required countries. How to build:
```
# For creating the custom build of libphonenumber-js lib
git clone https://github.com/EmesaDEV/libphonenumber-js
cd libphonenumber-js
npm i
# See the compiled lib in bundle/libphonenumber-js.min.js
# If you want to update the included countries list
# edit included_countries.js
npm run metadata:generate
npm run browser-build
```

More info: https://emesadev.atlassian.net/wiki/display/TPT/Creating+a+custom+build+for+libphonenumber-js+library

# libphonenumber-js

[![NPM Version][npm-badge]][npm]
[![Build Status][travis-badge]][travis]
[![Test Coverage][coveralls-badge]][coveralls]

A simpler (and smaller) rewrite of Google Android's famous `libphonenumber` library: easy phone number parsing and formatting in javascript.

[See Demo](https://halt-hammerzeit.github.io/libphonenumber-js/)

## LibPhoneNumber

[`libphonenumber`](https://github.com/googlei18n/libphonenumber) is a phone number formatting and parsing library released by Google, originally developed for (and currently used in) Google's [Android](https://en.wikipedia.org/wiki/Android_(operating_system)) mobile phone operating system. Implementing a rigorous phone number formatting and parsing library was crucial for the phone OS overall usability (back then, in the early 2000s, it was originally meant to be a phone after all, not just a SnapChat device).

`libphonenumber-js` is a simplified pure javascript port of the original `libphonenumber` library (written in C++ and Java because those are the programming languages used in Android OS). While `libphonenumber` has an [official javascript port](https://github.com/googlei18n/libphonenumber/tree/master/javascript) which is being maintained by Google, it is tightly coupled to Google's `closure` javascript utility framework. It still can be compiled into [one big bundle](http://stackoverflow.com/questions/18678031/how-to-host-the-google-libphonenumber-locally/) which weighs 220 KiloBytes — quite a size for a phone number input component. It [can be reduced](https://github.com/leodido/i18n.phonenumbers.js) to a specific set of countries only but that wouldn't be an option for a worldwide international solution.

One part of me was curious about how all this phone matching machinery worked, and another part of me was curious if there's a way to reduce those 220 KiloBytes to something more reasonable while also getting rid of the `closure` library and rewriting it all in pure javascript. So, that was my little hackathon for a couple of weeks, and seems that it succeeded. The resulting library does everything a modern web application needs while maintaining a much smaller size of about 75 KiloBytes.

## Difference from Google's `libphonenumber`

  * Pure javascript, doesn't require any 3rd party libraries
  * Metadata size is just about 75 KiloBytes while the original `libphonenumber` metadata size is about 200 KiloBytes
  * Better "as you type" formatting (and also more iPhone-alike style)
  * Doesn't parse alphabetic phone numbers like `1-800-GOT-MILK` as we don't use telephone sets in the XXIst century that much (and we have phonebooks in your mobile phones)
  * Doesn't handle carrier codes: they're only used in Colombia and Brazil, and only when dialing within those countries from a mobile phone to a fixed line number (the locals surely already know those carrier codes by themselves)
  * Assumes all phone numbers being `format`ted are internationally diallable, because that's the only type of phone numbers users are supposed to be inputting on websites (no one inputs short codes, emergency telephone numbers like `911`, etc.)
  * Doesn't parse phone numbers with extensions (again, this is not the type of phone numbers users should input on websites — they're supposed to input their personal mobile phone numbers, or home stationary phone numbers if they're living in an area where celltowers don't have a good signal, not their business/enterprise stationary phone numbers)
  * Doesn't use `possibleDigits` data to speed up phone number pre-validation (it just skips to the regular expression check itself)
  * Doesn't distinguish between fixed line, mobile, pager, voicemail, toll free and other XXth century bullsh*t
  * Doesn't format phone numbers for "out of country dialing", e.g. `011 ...` in the US (again, just use the `+...` notation accepted worldwide for mobile phones)
  * Doesn't parse `tel:...` URIs ([RFC 3966](https://www.ietf.org/rfc/rfc3966.txt)) because it's not relevant for user-facing web experience
  * When formatting international numbers replaces all braces, dashes, etc with spaces (because that's the logical thing to do, and leaving braces in an international number isn't)

## Installation

```
npm install libphonenumber-js --save
```

## Usage

```js
import { parse, format, asYouType } from 'libphonenumber-js'

parse('8 (800) 555 35 35', 'RU')
// { country: 'RU', phone: '8005553535' }

format('2133734253', 'US', 'International')
// '+1-213-373-4253'

new asYouType().input('+12133734')
// '+1 213 373 4'
new asYouType('US').input('2133734')
// '(213) 373-4'
```

## API

### parse(text, options)

`options` can be either an object

```js
country:
{
  restrict — (a two-letter country code)
             the phone number must be in this country

  default — (a two-letter country code)
            default country to use for phone number parsing and validation
            (if no country code could be derived from the phone number)
}
```

or just a two-letter country code which is gonna be `country.restrict`.

Returns `{ country, phone }` where `country` is a two-letter country code, and `phone` is a national (significant) number. If the phone number supplied isn't valid then an empty object `{}` is returned.

```js
parse('+1-213-373-4253') === { country: 'US', phone: '2133734253' }
parse('(213) 373-4253', 'US') === { country: 'US', phone: '2133734253' }
```

### format(parsed_number, format)

Formats a phone number using one of the following `format`s:
  * `International` — e.g. `+1 213 373 4253`
  * `International_plaintext` — (aka [`E.164`](https://en.wikipedia.org/wiki/E.164)) e.g. `+12133734253`
  * `National` — e.g. `(213) 373-4253`

`parsed_number` argument should be taken from the result of the `parse()` function call: `{ country, phone }`. `phone` must be a national (significant) number (i.e. no national prefix). `parsed_number` argument can also be expanded into two arguments:

```js
format({ country: 'US', phone: '2133734253' }, 'International') === '+1 213 373 4253'
format('2133734253', 'US', 'International') === '+1 213 373 4253'
```

### isValidNumber(number, country_code)

(aka `is_valid_number`)

This function is simply a wrapper for `parse`: if `parse` returns an empty object then the phone number is not valid.

```js
isValidNumber('+1-213-373-4253') === true
isValidNumber('+1-213-373') === false
isValidNumber('(213) 373-4253', 'US') === true
isValidNumber('(213) 37', 'US') === false
```

### `class` asYouType(default_country_code)

(aka `as_you_type`)

Creates a formatter for partially entered phone number. The two-letter `default_country_code` is optional and, if specified, is gonna be the default country for the phone number being input (in case it's not an international one). The instance of this class has two methods:

 * `input(text)` — takes any text and appends it to the input; returns the formatted phone number
 * `reset()` — resets the input

The instance of this class has also these fields:

 * `valid` — is the phone number being input a valid one already
 * `country` — a two-letter country code of the country this phone belongs to
 * `country_phone_code` — a phone code of the `country`
 * `template` — currently used phone number formatting template, where digits (and the plus sign, if present) are denoted by `x`-es

```js
new asYouType().input('+12133734') === '+1 213 373 4'
new asYouType('US').input('2133734') === '(213) 373-4'

const formatter = new asYouType()
formatter.input('+1-213-373-4253') === '+1 213 373 4253'
formatter.valid === true
formatter.country === 'US'
formatter.country_phone_code = '1'
formatter.template === 'xx xxx xxx xxxx'
```

## Metadata generation

Metadata is generated from Google's original [`PhoneNumberMetadata.xml`](https://github.com/googlei18n/libphonenumber/blob/master/resources/PhoneNumberMetadata.xml) by transforming XML into JSON and removing unnecessary fields.

How to update metadata:

  * Fork this repo
  * `npm install`
  * `npm run metadata:update`
  * Submit a pull request

To update metadata first it downloads the new [`PhoneNumberMetadata.xml`](https://github.com/googlei18n/libphonenumber/blob/master/resources/PhoneNumberMetadata.xml) into the project folder replacing the old one. Then it generates JSON metadata out of the XML one. After that it runs the tests and commits the new metadata.

<!-- ## To do

Everything's done -->

## Bug reporting

If you spot any inconsistencies with the [original Google's `libphonenumber`](https://libphonenumber.appspot.com/) then create an issue in this repo.

## Webpack

If you're using Webpack (which you most likely are) then make sure that

 * You have `json-loader` set up for `*.json` files in Webpack configuration
 * `json-loader` doesn't `exclude` `/node_modules/`
 * If you override `resolve.extensions` in Webpack configuration then make sure `.json` extension is present in the list

## Standalone

For those who aren't using bundlers for some reason there's a way to build a standalone version of the library

 * Clone the repo
 * `npm install`
 * `npm run browser-build`
 * See the `bundle` folder for `libphonenumber-js.min.js`

```html
<script src="/scripts/libphonenumber-js.min.js"></script>
<script>
  var libphonenumber = window['libphonenumber-js']
  alert(new libphonenumber.asYouType('US').input('213-373-4253'))
</script>
```

## Contributing

After cloning this repo, ensure dependencies are installed by running:

```sh
npm install
```

This module is written in ES6 and uses [Babel](http://babeljs.io/) for ES5
transpilation. Widely consumable JavaScript can be produced by running:

```sh
npm run build
```

Once `npm run build` has run, you may `import` or `require()` directly from
node.

After developing, the full test suite can be evaluated by running:

```sh
npm test
```

When you're ready to test your new functionality on a real project, you can run

```sh
npm pack
```

It will `build`, `test` and then create a `.tgz` archive which you can then install in your project folder

```sh
npm install [module name with version].tar.gz
```

## License

[MIT](LICENSE)
[npm]: https://www.npmjs.org/package/libphonenumber-js
[npm-badge]: https://img.shields.io/npm/v/libphonenumber-js.svg?style=flat-square
[travis]: https://travis-ci.org/halt-hammerzeit/libphonenumber-js
[travis-badge]: https://img.shields.io/travis/halt-hammerzeit/libphonenumber-js/master.svg?style=flat-square
[coveralls]: https://coveralls.io/r/halt-hammerzeit/libphonenumber-js?branch=master
[coveralls-badge]: https://img.shields.io/coveralls/halt-hammerzeit/libphonenumber-js/master.svg?style=flat-square
