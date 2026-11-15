---
layout: page
title: Tech Tricks
permalink: /tricks/
---
*If all else fails, reboot the machine*

# Shell
Cloning only a specific directory in a git repository, in this case the docs directory
``` sh
# git clone --no-checkout --bare <repository-url>
git clone -n --depth=1 --filter=tree:0
cd <repository-directory>
git sparse-checkout init
git sparse-checkout set docs
git checkout main
```

# JS
The following snippet converts an `ISO 3166-1 alpha-2` country code to its
emoji/[Regional indicator symbol](https://en.wikipedia.org/wiki/Regional_indicator_symbol) equivalent.
An example would converting from `EU` to 🇪🇺.

```js
function codeToFlag(code) {
	// notes: code should be an ISO3166 code or code such as ISO4217 which uses it.
	// 0x1F1E6 = REGIONAL INDICATOR SYMBOL LETTER A
	// 65 = 'A'
	return String.fromCodePoint(0x1F1E6 + code.codePointAt(0) - 65, 0x1F1E6 + code.codePointAt(1) - 65)
}
```

This doesn't work on Windows because Microsoft refuses to fix it.
A workaround is to use a font such as [country-flag-emoji-polyfill](https://npmjs.com/package/country-flag-emoji-polyfill)
