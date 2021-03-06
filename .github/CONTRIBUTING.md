# Contributing to nwb

## Choosing a Branch

`master` is used for critical fixes, documentation changes for the current version and any other changes which should be made available soon after being committed.

`next` is generally used for development of new features for the next major release and tracking non-critical dependency updates until the next release is ready.

## Type Checking

Use `npm run flow` to run [Flow](https://flow.org/) type checking.

The flow-for-vscode extension for VS Code is recommended if you're making changes to a file which is covered by Flow.

## Linting

Use `npm run lint` and `npm run lint:fix` - PRs which don't pass linting will fail.

See the [`eslint-config-jonnybuchanan` README](https://github.com/insin/eslint-config-jonnybuchanan#readme) for info on using nwb's lint config in your editor.

## Running Tests

- `npm test` will type check, lint, build and run all tests.

  This takes a few minutes to run and requires network access to install dependencies for tests which create projects.

  The last set of tests check that nwb exits correctly when tests fail, so you'll see *expected* error output at the end of the run. On a successful test run, the last line will be something along the lines of `147 passing (3m)`.

- `npm run test:coverage` is the same as the above, plus creation of a code coverage report.

  This is what's used for testing on Travis and Appveyor; code coverage results are posted from Travis to Coveralls after a successful build.

- `npm run test:watch` will watch files and run a subset of tests on every change, providing a quick check that you haven't broken any of the default config generation if you're working in that area.

  Command/project tests are too slow to run on each change.

## Building

- `npm run build` will type check, lint, and transpile code from `src/` to `lib/`.

- `npm run build:watch` will transpile code from `src/` to `lib/` on every change.

## Aliasing

The easiest way to run the development version of nwb is to alias the `bin` script from the nwb repo in the shell you'll be testing changes in:

```sh
# Bash etc.
alias nwb="node ~/repos/nwb/lib/bin/nwb.js"

# Cmder (aliases for Windows cmd.exe - http://cmder.net/)
alias nwb=node %USERPROFILE%\repos\nwb\lib\bin\nwb.js $*
```

> This uses the transpiled version, so don't forget to build beforehand or have `npm run build:watch` running.

Bumping the patch version in `package.json` to something silly provides a handy way to check if you're using an aliased version or the globally deployed version.

```sh
# Whoops, alias is missing or wrong
$ nwb v
v0.18.1

# Alias is set up correctly
$ nwb v
v0.18.999
```

## Implementation Details

nwb generates configuration for Babel, Webpack, Karma etc. on the fly.

The modules which handle generating each type of configuration provide a default set of features - and options for those features - if no further configuration is provided.

nwb commands (modules in `src/commands/`) provide configuration which controls which further features are used, or provides additional options. This is usually referred to as `buildConfig` throughout.

> Together, these provide a working, zero-config default for each command.

A user configuration file and command line arguments can be used to enable further features or tweak options. This configuration is usually referred to as `userConfig` throughout.

A concrete example of this is how nwb configures `html-webpack-plugin`:

```js
  if (buildConfig.html) {
    plugins.push(
      new HtmlPlugin({
        chunksSortMode: 'dependency',
        template: path.join(__dirname, '../templates/webpack-template.html'),
        ...buildConfig.html,
        ...userConfig.html,
      }),
    )
  }
```

1. Command configuration controls use of `HtmlPlugin` - only commands which relate to web apps will set this configuration and there's no way for users to accidentally turn this on when running commands it's not suitable for.

2. Working default configuration for the plugin is provided. `chunksSortMode` is essential to ensure script tags are created in the correct order and `template` is a bare-bones fallback which will work if the command or user doesn't configure a template.

3. Command configuration for this feature is then merged in to allow commands to change or add to this configuration as necessary.

  > Command-specific CLI flags and arguments usually affect creation of command configuration.

4. In this case we want to allow the user to specify their own template or use the other available options (e.g. `favicon`), so any user `webpack.html` config is merged over the top.

### Babel Config Generation - `src/createBabelConfig.js`

### Webpack Config Generation - `src/createWebpackConfig.js`

`createWebpackConfig()` takes command config and user config, and creates a Webpack config object - the rest of the module provides implementation details for different chunks of config.

`rules` config is generated by `createRules()` - this defines a mostly-static list of rules and loader which should suit most needs, but generates each's config in way which is completely configurable using an associated unique id. It uses `createStyleRules()` to handle the necessary differences between chaining loaders to handle stylesheets when serving vs. building and also to create additional rules for CSS preprocessor plugins when they're being used.

`plugin` config is generated by `createPlugins()` - this handles the necessary differences between development vs. production and serving vs. building. Additional plugins are enabled by configuration - usually command configuration is used to enable additional plugins and user config is used to allow users to override options, but there are no hard and fast rules here.

### Karma Config Generation - `src/createKarmaConfig.js`
