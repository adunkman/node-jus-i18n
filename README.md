Usage
=====

Default usage
-------------

    // Load module:
    var i18n = require('/path/to/i18n');
    // Optional: set default locale
    i18n.defaultLocale = 'fr';
    // Optional: add prefix and suffix around untranslated strings (default = '[T]' and '[/T]'
    i18n.debug();
    // Mandatory: load translation data
    i18n.load(
      catalogue, // Catalogue to load, undefined if you want to load default catalogue
      locales,   // array of locales to load, undefined if want to load all available locales
      function(errors, loadedLocales, store) { // Callback
        // errors = hash of exceptions thrown by each erroneous locale, or undefined if no error
        //          any global error will be stored in errors.ALL
        // loadedLocales = array of successfully loaded locales
        // store = store module
      }
    );
    // Go translate :)
    console.log(i18n.translate('Chicken')); // "Poulet"
    console.log(i18n.translate('Chicken', 'it')); // "Pollo"
    console.log(i18n.translate('Chicken {name}', {name: "KFC"})); // "Poulet KFC"
    console.log(i18n.translate('Chicken {name}', {name: "KFC"}, 'it')); // "Pollo KFC"

Integration with Express.js
---------------------------

    // ... app initialized ...
    // Load module:
    var i18n = require('/path/to/i18n');
    // Optional: configure default locale, debug mode, etc.
    // Then configure application:
    app.configure(function() {
      i18n.enableForApp(app, { // options (all are optional, you can pass {} or undefined
        "locale": "en",          // default locale
        "catalogue": "messages", // catalogue to load
        "locales": undefined,    // locales to load
      }, function(err) { // called when i18n has loaded messages
        ...
      });
    });
    // Your "req" object is augmented:
    req.i18n.translate(...)
    req.locales() // returns the list of user's accept-language, ordered by preference
    req.locale() // returns current user's chosen locale, stored in session if available
    // Your templates gain new helpers:
    ...<%= _('Hello, {name}', {name: userName}) %>...
    ...<%= plural('You have {n} messages', nbMessages) %>...

Store your messages
===================

There is only one messages store currently supported is "module", which means you store your messages as a Node.js module.

Store: module
-------------

A whole catalogue in a single file:

    // Module name: "./i18n-data/%catalogue%"
    // ./i18n-data/messages.js
    module.exports = {
      "fr": { "Chicken": "Poulet", "Chicken %name%": "Poulet %name%" },
      "it": { "Chicken": "Pollo", "Chicken %name%": "Pollo %name%" },
    };
    // will be loaded with i18n.load('messages');

Or split by locale:

    // Module name: "./i18n-data/%catalogue%/%locale%"
    // ./i18n-data/messages/fr.js
    module.exports = { "Chicken": "Poulet", "Chicken %name%": "Poulet %name%" };
    // ./i18n-data/messages/it.js
    module.exports = { "Chicken": "Pollo", "Chicken %name%": "Pollo %name%" };
    // will be loaded with i18n.load('messages', ['fr', 'it']);

Note that you can customize the path to i18n-data modules:

    i18n.i18nDataModuleName.__default__ = process.cwd() + "/i18n-data";

Store: file
-----------

Soon available (format: ini).

Store: db
---------

Soon available (redis, mongodb, mysql...).

Plural forms
============

This is the most important feature to come. Still in development though.

Planned API (WiP)
-----------------

The base function will expect only the "plural form", and the associated number:

    plural(msg, number)

A "plural form" could be a simple string, or a complex structure.
Here are some valid formats we could imagine:

    // Simple string
    "[0]No message|[1]One message|[2,+Inf)%count% messages"

    // Complex structure
    [
      [ function(count){return count == 0;}, "No message" ],
      [ function(count){return count == 1;}, "One message" ],
      [ function(count){return count >= 2;}, "%count% messages" ],
    ]

Supporting both type of structures could ease support for complex rules like polish plurals (where we need to use euclidian divides). Eval() is an option too...

About translating message passer to `plural()`, following behavior has to be discussed:

> "plural()" is able to automatically translate the message, BUT ONLY IF YOU EXPECT IT TO:
> 
> * plural(msg, number) → will not translate msg
> * plural(msg, number, params, locale, catalogue) → will translate msg
> 
> If you want "plural()" to translate the message, but using default locale and no replacement, then call `plural(msg, number, {})`
> 
> Other option: always translate.

Example in a template:

    <%= plural("You have %n% messages", 3, {}) %>
    // _("You have %n% messages") returns "[0]No message|[1]One message|[2,+Inf)%n% messages"
    // plural("[0]No message|[1]One message|[2,+Inf)%n% messages", 3) returns "3 messages"

Contextual translations
=======================

You may sometimes need to translate a sentence differently depending on a unpredictible context. Usual case is the gender (male/female).
This is handled using a special parameter named "context", and a special translation "context:message".

For example, supposing you want to say "hello, {name}" differently depending on civility ("mr", "mrs", "miss"), you will provide these translations in the store:

    {
      "hello, {name}": "hello, {name}",                  // default translation, no context
      "mr:hello, {name}": "hello, Mister {name}",        // translation for civility "mr"
      "mrs:hello, {name}": "hello, Mrs. {name}",         // translation for civility "mrs"
      "miss:hello, {name}": "hello, Miss {name}"         // translation for civility "miss"
    }

You will then be able to translate "hello, {name}" differently depending on provided context:

    i18n.translate("hello, {name}", {name: "Jones", context: "mr"});  // hello, Mister Jones
    i18n.translate("hello, {name}", {name: "Jones", context: "mrs"}); // hello, Mrs. Jones

Configuration
=============

* Customize the session key to store user's locale:
  
      i18n.localeSessKey = 'locale';

* Customize the messages store:
  
      // Embedded store
      i18n.setStore('module', options, function(err, i18n) {
        ...
      });
      // You custom store module
      i18n.setStore(require('/path/to/my/store'), options, function(err, i18n) {
        ...
      });
  
  Beware you must call "i18n.load(...)" again if you had already loaded another store.
  You can use only one store at a time.

* Customize default locale:
  
      i18n.defaultLocale = 'en';

* Default catalogue to load and search translations from:
  
      i18n.defaultCatalogue = 'messages';

* Change format of replaced parameters in your messages:
  
      i18n.replaceFormat = '{...}';
      // i18n.replaceFormat = ':...';
      // and i18n.translate('hello, :name', {name: 'John'}) will work as expected

* In plural forms, the parameter 'n' is replaced by the number, you can change this name:
  
      i18n.defaultPluralReplace = 'n';

Write your own store
--------------------

You must write a module that will expose at least two self-explanatory methods:
* load(catalogue, locales, i18n, callback)
  * catalogue and i18n will always be provided by i18n module.
  * if no locale is provided, you're expected to enable all available locales.
  * callback expects following parameters: (errors, loadedLocales, this)
* get(key, locale, catalogue, i18n)
  * all parameters will always be provided by i18n module.
  * if no translation is found, you MUST return undefined.
  * this function HAS TO BE synchronous.
* locales(prefix, callback)
  * callback expects following parameters: (err, array of locales starting with prefix, this)
* catalogues(callback)
  * callback expects following parameters: (err, array of available catalogues, this)
* configure(options, callback)
  * configure the store, options depend on your store
  * callback expects following parameters: (err, this)

The i18n module is passed to these methods whenever you would need it in your store.

Stupid example (will always translate "js", and only this one, into "rox"):

    var data = {};
    exports.load = function(catalogue, locale, i18n, callback) {
      catalogue = catalogue || i18n.defaultCatalogue;
      locale = locale || i18n.defaultLocale;
      if (!data[catalogue]) {
        data[catalogue] = {locale: {"js": "rox"}};
      } else {
        if (!data[catalogue][locale]) {
          data[catalogue][locale] = {"js": "rox"};
        } else {
          data[catalogue][locale]["js"] = "rox";
        }
      }
      // Make it asynchronous
      setTimeout(function() { callback(undefined, i18n, exports); }, 1);
    }
    exports.get = function(key, locale, catalogue) {
      return data[catalogue][locale][key];
    }
    exports.configure = function(options, callback) {
      // This time, it's synchronous, your implementation, your choice
      callback(undefined, i18n, this);
    }
      

TODO
====

* Fix the data loading when we specify the list of loaded locales.
* Provide more stores (at least Redis).
* Better documentation.
* If provided a "file" store: Ability to merge data from more than one folder.
* Ability to use more than one store at same time.
* Better support for locales "lang_COUNTRY" (loads messages for locales "lang" and "lang-country")
* All these things I didn't think about yet.


* Done: <del>Plural forms, including ranges and expressions recognition</del>.
* Done: <del>Context like gender (original idea from dialect)</del>.
