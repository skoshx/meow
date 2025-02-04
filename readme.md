# meow

> CLI app helper

![](meow.gif)

*I would recommend reading this [guide](https://clig.dev) on how to make user-friendly command-line tools.*

## Features

- Parses arguments
- Converts flags to [camelCase](https://github.com/sindresorhus/camelcase)
- Negates flags when using the `--no-` prefix
- Outputs version when `--version`
- Outputs description and supplied help text when `--help`
- Makes unhandled rejected promises [fail hard](https://github.com/sindresorhus/hard-rejection) instead of the default silent fail
- Sets the process title to the binary name defined in package.json

## Install

```
$ npm install meow
```

## Usage

```
$ ./foo-app.js unicorns --rainbow
```

```js
#!/usr/bin/env node
import meow from 'meow';
import foo from './lib/index.js';

const cli = meow(`
	Usage
	  $ foo <input>

	Options
	  --rainbow, -r  Include a rainbow

	Examples
	  $ foo unicorns --rainbow
	  🌈 unicorns 🌈
`, {
	importMeta: import.meta,
	flags: {
		rainbow: {
			type: 'boolean',
			alias: 'r'
		}
	}
});
/*
{
	input: ['unicorns'],
	flags: {rainbow: true},
	...
}
*/

foo(cli.input[0], cli.flags);
```

## API

### meow(helpText, options?)
### meow(options)

Returns an `object` with:

- `input` *(Array)* - Non-flag arguments
- `flags` *(Object)* - Flags converted to camelCase excluding aliases
- `commands` *(Object)* - Subcommands with values parsed with respect to subcommand's meow instance.
- `unnormalizedFlags` *(Object)* - Flags converted to camelCase including aliases
- `pkg` *(Object)* - The `package.json` object
- `help` *(string)* - The help text used with `--help`
- `showHelp([exitCode=2])` *(Function)* - Show the help text and exit with `exitCode`
- `showVersion()` *(Function)* - Show the version text and exit

#### helpText

Type: `string`

Shortcut for the `help` option.

#### options

Type: `object`

##### importMeta

Type: `object`

Pass in [`import.meta`](https://nodejs.org/dist/latest/docs/api/esm.html#esm_import_meta). This is used to find the correct package.json file.

##### flags

Type: `object`

Define argument flags.

The key is the flag name in camel-case and the value is an object with any of:

- `type`: Type of value. (Possible values: `string` `boolean` `number`)
- `alias`: Usually used to define a short flag alias.
- `default`: Default value when the flag is not specified.
- `isRequired`: Determine if the flag is required. (Default: false)
	- If it's only known at runtime whether the flag is required or not, you can pass a `Function` instead of a `boolean`, which based on the given flags and other non-flag arguments, should decide if the flag is required. Two arguments are passed to the function:
	- The first argument is the **flags** object, which contains the flags converted to camel-case excluding aliases.
	- The second argument is the **input** string array, which contains the non-flag arguments.
	- The function should return a `boolean`, true if the flag is required, otherwise false.
- `isMultiple`: Indicates a flag can be set multiple times. Values are turned into an array. (Default: false)
	- Multiple values are provided by specifying the flag multiple times, for example, `$ foo -u rainbow -u cat`. Space- or comma-separated values are [currently *not* supported](https://github.com/sindresorhus/meow/issues/164).

Note that flags are always defined using a camel-case key (`myKey`), but will match arguments in kebab-case (`--my-key`).

Example:

```js
flags: {
	unicorn: {
		type: 'string',
		alias: 'u',
		default: ['rainbow', 'cat'],
		isMultiple: true,
		isRequired: (flags, input) => {
			if (flags.otherFlag) {
				return true;
			}

			return false;
		}
	}
}
```

##### commands

Type: `object`

Define subcommands. Subcommands don't actually call any commands for you, it only takes care of parsing the subcommand flags, inputs, and showing the subcommand helptext.

The key is the name of the subcommand and the value is a function that returns an instance of `meow`.

The following values get passed to the subcommand function:
	- `options`: The options from the parent `meow` instance.

Example:

```js
const commands = {
	subcommand = (options) => meow({
		...options,
		description: 'Subcommand description',
		help: `
			Unicorn command
			Usage:
				foo unicorn <input>
		`,
		flags: {
			unicorn: {alias: 'u', isRequired: true},
		},
	});
};
const cli = meow({
	importMeta: import.meta,
	description: 'Custom description',
	help: `
		Usage
			foo unicorn <input>
	`,
	commands: {
		unicorn: commands.subcommand,
	},
	flags: {},
});

// call subcommand
const [command, parsedCli] = Object.entries(cli.commands ?? {})?.[0] ?? [];
// command => "unicorn"
// parsedCli => parsed meow instance of unicorn subcommand
commands[command](parsedCli);
```

##### description

Type: `string | boolean`\
Default: The package.json `"description"` property

Description to show above the help text.

Set it to `false` to disable it altogether.

##### help

Type: `string | boolean`

The help text you want shown.

The input is reindented and starting/ending newlines are trimmed which means you can use a [template literal](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/template_strings) without having to care about using the correct amount of indent.

The description will be shown above your help text automatically.

##### version

Type: `string | boolean`\
Default: The package.json `"version"` property

Set a custom version output.

##### autoHelp

Type: `boolean`\
Default: `true`

Automatically show the help text when the `--help` flag is present. Useful to set this value to `false` when a CLI manages child CLIs with their own help text.

This option is only considered when there is only one argument in `process.argv`.

##### autoVersion

Type: `boolean`\
Default: `true`

Automatically show the version text when the `--version` flag is present. Useful to set this value to `false` when a CLI manages child CLIs with their own version text.

 This option is only considered when there is only one argument in `process.argv`.

##### pkg

Type: `object`\
Default: Closest package.json upwards

package.json as an `object`.

*You most likely don't need this option.*

##### argv

Type: `string[]`\
Default: `process.argv.slice(2)`

Custom arguments object.

##### inferType

Type: `boolean`\
Default: `false`

Infer the argument type.

By default, the argument `5` in `$ foo 5` becomes a string. Enabling this would infer it as a number.

##### booleanDefault

Type: `boolean | null | undefined`\
Default: `false`

Value of `boolean` flags not defined in `argv`.

If set to `undefined`, the flags not defined in `argv` will be excluded from the result.
The `default` value set in `boolean` flags take precedence over `booleanDefault`.

_Note: If used in conjunction with `isMultiple`, the default flag value is set to `[]`._

__Caution: Explicitly specifying `undefined` for `booleanDefault` has different meaning from omitting key itself.__

Example:

```js
import meow from 'meow';

const cli = meow(`
	Usage
	  $ foo

	Options
	  --rainbow, -r  Include a rainbow
	  --unicorn, -u  Include a unicorn
	  --no-sparkles  Exclude sparkles

	Examples
	  $ foo
	  🌈 unicorns✨🌈
`, {
	importMeta: import.meta,
	booleanDefault: undefined,
	flags: {
		rainbow: {
			type: 'boolean',
			default: true,
			alias: 'r'
		},
		unicorn: {
			type: 'boolean',
			default: false,
			alias: 'u'
		},
		cake: {
			type: 'boolean',
			alias: 'c'
		},
		sparkles: {
			type: 'boolean',
			default: true
		}
	}
});
/*
{
	flags: {
		rainbow: true,
		unicorn: false,
		sparkles: true
	},
	unnormalizedFlags: {
		rainbow: true,
		r: true,
		unicorn: false,
		u: false,
		sparkles: true
	},
	…
}
*/
```

##### hardRejection

Type: `boolean`\
Default: `true`

Whether to use [`hard-rejection`](https://github.com/sindresorhus/hard-rejection) or not. Disabling this can be useful if you need to handle `process.on('unhandledRejection')` yourself.

##### allowUnknownFlags

Type `boolean`\
Default: `true`

Whether to allow unknown flags or not.

## Promises

Meow will make unhandled rejected promises [fail hard](https://github.com/sindresorhus/hard-rejection) instead of the default silent fail. Meaning you don't have to manually `.catch()` promises used in your CLI.

## Tips

See [`chalk`](https://github.com/chalk/chalk) if you want to colorize the terminal output.

See [`get-stdin`](https://github.com/sindresorhus/get-stdin) if you want to accept input from stdin.

See [`conf`](https://github.com/sindresorhus/conf) if you need to persist some data.

See [`update-notifier`](https://github.com/yeoman/update-notifier) if you want update notifications.

[More useful CLI utilities…](https://github.com/sindresorhus/awesome-nodejs#command-line-utilities)

---

<div align="center">
	<b>
		<a href="https://tidelift.com/subscription/pkg/npm-meow?utm_source=npm-meow&utm_medium=referral&utm_campaign=readme">Get professional support for this package with a Tidelift subscription</a>
	</b>
	<br>
	<sub>
		Tidelift helps make open source sustainable for maintainers while giving companies<br>assurances about security, maintenance, and licensing for their dependencies.
	</sub>
</div>
