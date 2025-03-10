These directions are for converting an _existing_ Ember app to TypeScript. If you are starting a new app, you can use the directions in [Getting Started][].

## Enable TypeScript Features

### Install TypeScript and Related Packages

See [Getting Started: Packages to Support TypeScript][packages] for descriptions of these packages.

```shell
npm add --save-dev typescript @tsconfig/ember
npm add --save-dev @types/qunit @types/rsvp
npm add --save-dev @typescript-eslint/eslint-plugin @typescript-eslint/parser
npm remove @babel/plugin-proposal-decorators @babel/eslint-parser
```

### Add TypeScript Configuration

Add a `tsconfig.json` file to the root of your project. Copy its contents from the [current output from the Ember CLI blueprints][tsconfig.json].

### Set Up TypeScript for EmberData

Follow the instructions in the [EmberData Typescript Guides][ED-ts-guides].

### Enable TypeScript Transpilation for Builds

To enable TypeScript transpilation in your app, simply add the corresponding configuration for Babel to your `ember-cli-build.js` file.

```javascript {data-filename="ember-cli-build.js" data-diff="+3"}
module.exports = function (defaults) {
  const app = new EmberApp(defaults, {
    "ember-cli-babel": { enableTypeScriptTransform: true },
    // ...
  });

  return app.toTree();
};
```

### Enable Type Checking in CI

To easily check types with the command line, add the `lint:types` script as shown [here][lint-types].

The default `lint` script generated by Ember CLI will include the `lint:types` script automatically.

### Configure Blueprint Generators to Use TypeScript

With the following configuration, project files will be generated with `.ts` extensions instead of `.js`:

```javascript {data-filename=".ember-cli" data-diff="-2,+3"}
{
  "isTypeScriptProject": false,
  "isTypeScriptProject": true,
}
```

```js {data-filename="config/ember-cli-update.json" data-diff="+12"}
{
  // ...
  "packages": [
    {
      "name": "ember-cli",
      // ...
      "blueprints": [
        {
          // ...
          "options": [
            // ...
            "--typescript"
          ]
        }
      ]
    }
  ]
}
```

### Configure ESLint

Then, update your `eslint.config.mjs` to include the [current output from the Ember CLI blueprints][eslintrc]. You might consider using ESLint [overrides][] configuration to separately configure your JavaScript and TypeScript files during the migration.

### Add Initial Type Declarations

Add types for your `config/environment.js` file by creating a type declaration file at `app/config/environment.d.ts`. You can find an example file in the [current output from the Ember CLI blueprints][environment.d.ts].

## Migrate Existing Code to TypeScript

Once you have set up TypeScript following the guides above, you can begin to migrate your files incrementally by changing their extensions from `.js` to `.ts`. Fortunately, TypeScript allows for gradual typing. This means that you can use TypeScript and JavaScript files interchangeably, so you can convert your app piecemeal.

Some specific tips for success on the technical front:

### Strictness

Use the [_strictest_ strictness][strictness] settings that our typings allow. While it may be tempting to start with the _loosest_ strictness settings and then to tighten them down as you go, this will actually mean that "getting your app type-checking" will become a repeated process—getting it type-checking with every new strictness setting you enable—rather than something you do just once.

### Gradual Typing Hacks

Many of your files might reference types in other files that haven't been converted yet. There are several strategies you can employ to avoid a chain-reaction resulting in having to convert your entire app at once:

The [`unknown`][unknown] type—You can sometimes get pretty far just by annotating types as `unknown`. If `unknown` is too wide of a type, however, you'll need a more robust solution.

[TypeScript declaration files][dts] (`.d.ts`)—These files are a straightforward way to document TypeScript types for JavaScript files without converting them. One downside of declaration files, however, is that they can easily get out-of-sync with the corresponding JavaScript file, so we only recommend this option as a temporary step.

[JSDoc][] and [`allowJs`][allowJs]—Another way to document TypeScript types for JavaScript files without converting them is to add JSDoc "type hints" to the files and enable the `allowJs` compiler option in your `tsconfig.json`. While the JSDoc type syntax can be a bit cumbersome, it is much more likely to stay in sync. You can even type-check your JavaScript files using the [`@ts-check`][ts-check] directive.

The [`any`][any] type—Opt out of type checking altogether for a value by annotating it as `any`.

The [`@ts-expect-error`][ts-expect-error] directive—A better strategy than `any`, however, is to mark offending parts of your code with a `@ts-expect-error` directive. This comment will ignore a type-checking error and allow the TypeScript compiler to assume that the value is of the type `any`. If the code stops triggering the error, TypeScript will let you know.

### Outer Leaves First

A good approach to gradual typing is to start at your outer "leaf" modules (the ones that don't import anything else from your app, only from Ember or third-party libraries) and then work your way "inward" (toward the modules with many internal imports). Often the highest-value modules are your EmberData models and any core services that are used everywhere else in the app–and those are also the ones that tend to have the most cascading effects (having to update _tons_ of other places in your app) when you type them later in the process. By starting with the outer leaves, you won't have to use as many of our gradual typing hacks.

### Prefer Octane Idioms

In general, we recommend migrating to Octane idioms before, or in conjunction with, your migration to TypeScript. See ["Working With Ember Classic"][legacy] for more details.

## ember-cli-typescript

The `ember-cli-typescript` package was used to add TypeScript support to Ember apps before Ember's native TypeScript support was available. You do _not_ need `ember-cli-typescript` installed for new apps or addons.

If you're migrating from `ember-cli-typescript` to Ember's native TypeScript support, most of your existing configuration will still be relevant. Just read through the steps of this guide and ensure that your config matches the expected config as described above.

<!-- Internal links -->

[ED-ts-guides]: ../../core-concepts/ember-data/#toc_adding-emberdata-types-to-an-existing-typescript-app
[getting started]: ../../getting-started/
[legacy]: ../../additional-resources/legacy/
[packages]: ../../getting-started/#toc_packages-to-support-typescript
[strictness]: ../../additional-resources/faq/#toc_strictness

<!-- External links -->

[allowJs]: https://www.typescriptlang.org/tsconfig/#allowJs
[any]: https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#any
[dts]: https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html
[environment.d.ts]: https://github.com/ember-cli/editor-output/blob/stackblitz-app-output-typescript/app/config/environment.d.ts
[eslintrc]: https://github.com/ember-cli/editor-output/blob/stackblitz-app-output-typescript/eslint.config.mjs
[lint-types]: https://github.com/ember-cli/editor-output/blob/stackblitz-app-output-typescript/package.json
[JSDoc]: https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html#handbook-content
[overrides]: https://eslint.org/docs/latest/use/configure/configuration-files#configuration-based-on-glob-patterns
[ts-check]: https://www.typescriptlang.org/docs/handbook/intro-to-js-ts.html#ts-check
[ts-expect-error]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-9.html
[tsconfig.json]: https://github.com/ember-cli/editor-output/blob/stackblitz-app-output-typescript/tsconfig.json
[unknown]: https://www.typescriptlang.org/docs/handbook/2/functions.html
