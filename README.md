[![Build Status](https://travis-ci.org/ToonvanStrijp/nestjs-i18n.svg?branch=master)](https://travis-ci.org/ToonvanStrijp/nestjs-i18n) [![Greenkeeper badge](https://badges.greenkeeper.io/ToonvanStrijp/nestjs-i18n.svg)](https://greenkeeper.io/)
[![Coverage Status](https://coveralls.io/repos/github/ToonvanStrijp/nestjs-i18n/badge.svg?branch=master)](https://coveralls.io/github/ToonvanStrijp/nestjs-i18n?branch=master)

## Description

The **i18n** module for [Nest](https://github.com/nestjs/nest).

## Installation

```bash
$ npm i --save nestjs-i18n
```

## Versions

To keep your setup working use the correct version of `nestjs-i18n`.

| nestjs-i18n version    | nestjs version |
| ---------------------- | -------------- |
| **v7.0.0** or greather | **v7.0.0**     |
| **v6.0.0** or below    | **v6.0.0**     |

## Quick Start

Build in we have a JSON parser (`I18nJsonParser`) this parser handles to following structure

### Structure

create a directory and in it define your language keys as directories.

```
i18n
├── en
│   ├── category.json
│   └── auth.json
└── nl
    ├── category.json
    └── auth.json
```

### Translation File

The format of a translation file could look like this:

```json
{
  "HELLO_MESSAGE": "Hello {username}",
  "GOODBYE_MESSAGE": "Goodbye {username}",
  "USER_ADDED_PRODUCT": "{0.username} added {1.productName} to cart",
  "SETUP": {
    "WELCOME": "Welcome {0.username}",
    "GOODBYE": "Goodbye {0.username}"
  },
  "ARRAY": ["ONE", "TWO", "THREE"]
}
```

String formatting is done by: [string-format](https://github.com/davidchambers/string-format)

### Translation Module

To use the translation service we first add the module. **The `I18nModule` has a `@Global()` attribute so you should only import it once**.

```typescript
import { Module } from '@nestjs/common';
import * as path from 'path';
import { I18nModule } from 'nestjs-i18n';

@Module({
  imports: [
    I18nModule.forRoot({
      fallbackLanguage: 'en',
      parser: I18nJsonParser,
      parserOptions: {
        path: path.join(__dirname, '/i18n/'),
      },
    }),
  ],
  controllers: [],
})
export class AppModule {}
```

#### Using forRootAsync()

```typescript
import { Module } from '@nestjs/common';
import * as path from 'path';
import { I18nModule } from 'nestjs-i18n';

@Module({
  imports: [
    I18nModule.forRootAsync({
      useFactory: (configService: ConfigurationService) => ({
        fallbackLanguage: configService.fallbackLanguage, // e.g., 'en'
        parserOptions: {
          path: path.join(__dirname, '/i18n/'),
        },
      }),
      parser: I18nJsonParser,
      inject: [ConfigurationService],
    }),
  ],
  controllers: [],
})
export class AppModule {}
```

## Live reloading / Refreshing translations

To use live reloading use the `watch` option in the `I18nJsonParser`. The `I18nJsonParser` watches the `i18n` folder for changes and when needed updates the `translations` or `languages`.

```typescript
I18nModule.forRoot({
  fallbackLanguage: 'en',
  parser: I18nJsonParser,
  parserOptions: {
    path: path.join(__dirname, '/i18n/'),
    // add this to enable live translations
    watch: true,
  },
});
```

To refresh your translations and languages manually:

```typescript
await this.i18nService.refresh();
```

### Parser

A default JSON parser (`I18nJsonParser`) is included.

To implement your own `I18nParser` take a look at this example [i18n.json.parser.ts](https://github.com/ToonvanStrijp/nestjs-i18n/blob/master/src/lib/parsers/i18n.json.parser.ts).

#### Live translations / languages

To provide live translations you can return an observable within the extended `I18nParser` class. For and implementation example you can take a look at the [i18n.json.parser.ts](https://github.com/ToonvanStrijp/nestjs-i18n/blob/master/src/lib/parsers/i18n.json.parser.ts).

```typescript
export class I18nMysqlParser extends I18nParser {
  constructor(
    @Inject(I18N_PARSER_OPTIONS)
    private options: I18nJsonParserOptions,
  ) {
    super();
  }

  async languages(): Promise<string[] | Observable<string[]>> {
    // for example do a database call here
    return observableOf(['nl', 'en']);
  }

  async parse(): Promise<I18nTranslation | Observable<I18nTranslation>> {
    // for example do a database call here
    return observableOf({
      nl: {
        HELLO: 'Hallo',
      },
      en: {
        HELLO: 'Hello',
      },
    });
  }
}
```

### Language Resolvers

To make it easier to manage in what language to respond you can make use of resolvers

```typescript
@Module({
  imports: [
    I18nModule.forRoot({
      fallbackLanguage: 'en',
      parser: I18nJsonParser,
      parserOptions: {
        path: path.join(__dirname, '/i18n/'),
      },
      resolvers: [
        { use: QueryResolver, options: ['lang', 'locale', 'l'] },
        new HeaderResolver(['x-custom-lang']),
        AcceptLanguageResolver,
        new CookieResolver(['lang', 'locale', 'l']),
      ],
    }),
  ],
  controllers: [HelloController],
})
export class AppModule {}
```

Currently, there are four built-in resolvers

| Resolver                 | Default value |
| ------------------------ | ------------- |
| `QueryResolver`          | `none`        |
| `HeaderResolver`         | `none`        |
| `AcceptLanguageResolver` | `N/A`         |
| `CookieResolver`         | `lang`        |

To implement your own resolver (or custom logic) use the `I18nResolver` interface. The resolvers are provided via the nestjs dependency injection, this way you can inject your own services if needed.

```typescript
@Injectable()
export class QueryResolver implements I18nResolver {
  constructor(@I18nResolverOptions() private keys: string[]) {}

  resolve(req: any) {
    let lang: string;

    for (const key of this.keys) {
      if (req.query != undefined && req.query[key] !== undefined) {
        lang = req.query[key];
        break;
      }
    }

    return lang;
  }
}
```

To provide initial options to your custom resolver use the `@I18nResolverOptions()` decorator, also provide the resolver as followed:

```typescript
I18nModule.forRoot({
  fallbackLanguage: 'en',
  parser: I18nJsonParser,
  parserOptions: {
    path: path.join(__dirname, '/i18n/'),
  },
  resolvers: [{ use: QueryResolver, options: ['lang', 'locale', 'l'] }],
});
```

#### Using forRootAsync()

```typescript
I18nModule.forRootAsync({
  useFactory: () => {
    return {
      fallbackLanguage: 'en',
      parserOptions: {
        path: path.join(__dirname, '/i18n'),
      },
    };
  },
  parser: I18nJsonParser,
  resolvers: [{ use: QueryResolver, options: ['lang', 'locale', 'l'] }],
});
```

### Translating with i18n module

#### `I18nLang` decorator and `I18nService`

```typescript
@Controller()
export class SampleController {
  constructor(private readonly i18n: I18nService) {}

  @Get()
  async sample(@I18nLang() lang: string) {
    await this.i18n.translate('HELLO_MESSAGE', {
      lang: lang,
      args: { id: 1, username: 'Toon' },
    });
    await this.i18n.translate('SETUP.WELCOME', {
      lang: 'en',
      args: { id: 1, username: 'Toon' },
    });
    await this.i18n.translate('ARRAY.0', { lang: 'en' });
  }
}
```

#### `I18n` decorator

```typescript
@Controller()
export class SampleController {
  @Get()
  async sample(@I18n() i18n: I18nContext) {
    await i18n.translate('HELLO_MESSAGE', {
      args: { id: 1, username: 'Toon' },
    });
    await i18n.translate('SETUP.WELCOME', {
      args: { id: 1, username: 'Toon' },
    });
    await i18n.translate('ARRAY.0');
  }
}
```

No need to handle `lang` manually.

#### `I18nRequestScopeService` within a custom service using request scoped translation service

```typescript
@Injectable()
export class SampleService {
  constructor(private readonly i18n: I18nRequestScopeService) {}

  async doFancyStuff() {
    await this.i18n.translate('HELLO_MESSAGE', {
      args: { id: 1, username: 'Toon' },
    });
    await this.i18n.translate('SETUP.WELCOME', {
      args: { id: 1, username: 'Toon' },
    });
    await this.i18n.translate('ARRAY.0');
  }
}
```

To be used within other services like sending E-mails.
The advantage is that you don't have to worry about transporting `lang` from the `Request` to your service.

**Use with caution!** The `I18nRequestScopeService` uses the `REQUEST` scope and is no singleton.
This will be inherited to all consumers of `I18nRequestScopeService`!
Read [Nest Docs](https://docs.nestjs.com/fundamentals/injection-scopes) for more information.

**Dont use `I18nRequestScopeService` within controllers.** The `I18n` decorator is a much better solution.

# CLI

To easily check if your i18n folder is correctly structured you can use the following command:
`nest-i18n check <i18n-path>`

example: `nest-i18n check src/i18n`

This is very useful inside a CI environment to prevent releases with missing translations.

# Breaking changes:

- from V6.0.0 on we implemented the `I18nParser`, by using this we can easily support different formats other than JSON. To migrate to this change look at the [Quick start](#quick-start) above. There are some changes in the declaration of the `I18nModule`. Note: the translate function returns a Promise<string>. So you need to call it using await i18n.translate('HELLO');

- from V4.0.0 on we changed the signature of the `translate` method, the language is now optional, if no language is given it'll fallback to the `fallbackLanguage`

- from V3.0.0 on we load translations based on their directory name instead of file name. Change your translations files to the structure above: [info](https://github.com/ToonvanStrijp/nestjs-i18n#structure)
