---
id: how-to-guides
title: Search How-To guides
sidebar_label: How-To guides
description: Search How To guides
---

## How to implement your own Search API

The Search plugin provides implementation of one primary API by default: the
[SearchApi](https://github.com/backstage/backstage/blob/db2666b980853c281b8fe77905d7639c5d255f13/plugins/search/src/apis.ts#L35),
which is responsible for talking to the search-backend to query search results.

There may be occasions where you need to implement this API yourself, to
customize it to your own needs - for example if you have your own search backend
that you want to talk to. The purpose of this guide is to walk you through how
to do that in two steps.

1. Implement the `SearchApi`
   [interface](https://github.com/backstage/backstage/blob/db2666b980853c281b8fe77905d7639c5d255f13/plugins/search/src/apis.ts#L31)
   according to your needs.

```typescript
export class SearchClient implements SearchApi {
  // your implementation
}
```

2. Override the API ref `searchApiRef` with your new implemented API in the
   `App.tsx` using `ApiFactories`.
   [Read more about App APIs](https://backstage.io/docs/api/utility-apis#app-apis).

```typescript
const app = createApp({
  apis: [
    // SearchApi
    createApiFactory({
      api: searchApiRef,
      deps: { discovery: discoveryApiRef },
      factory({ discovery }) {
        return new SearchClient({ discoveryApi: discovery });
      },
    }),
  ],
});
```

## How to index TechDocs documents

The TechDocs plugin has supported integrations to Search, meaning that it
provides a default collator factory ready to be used.

The purpose of this guide is to walk you through how to register the
[DefaultTechDocsCollatorFactory](https://github.com/backstage/backstage/blob/master/plugins/techdocs-backend/src/search/DefaultTechDocsCollatorFactory.ts)
in your App, so that you can get TechDocs documents indexed.

If you have been through the
[Getting Started with Search guide](https://backstage.io/docs/features/search/getting-started),
you should have the `packages/backend/src/plugins/search.ts` file available. If
so, you can go ahead and follow this guide - if not, start by going through the
getting started guide.

1. Import the `DefaultTechDocsCollatorFactory` from
   `@backstage/plugin-techdocs-backend`.

```typescript
import { DefaultTechDocsCollatorFactory } from '@backstage/plugin-techdocs-backend';
```

2. If there isn't an existing schedule you'd like to run the collator on, be
   sure to create it first. Something like...

```typescript
import { Duration } from 'luxon';

const every10MinutesSchedule = env.scheduler.createScheduledTaskRunner({
  frequency: Duration.fromObject({ seconds: 600 }),
  timeout: Duration.fromObject({ seconds: 900 }),
  initialDelay: Duration.fromObject({ seconds: 3 }),
});
```

3. Register the `DefaultTechDocsCollatorFactory` with the IndexBuilder.

```typescript
indexBuilder.addCollator({
  schedule: every10MinutesSchedule,
  factory: DefaultTechDocsCollatorFactory.fromConfig(env.config, {
    discovery: env.discovery,
    logger: env.logger,
    tokenManager: env.tokenManager,
  }),
});
```

You should now have your TechDocs documents indexed to your search engine of
choice!

If you want your users to be able to filter down to the techdocs type when
searching, you can update your `SearchPage.tsx` file in
`packages/app/src/components/search` by adding `techdocs` to the list of values
of the `SearchType` component.

```tsx
<Paper className={classes.filters}>
  <SearchType
    values={['techdocs', 'software-catalog']}
    name="type"
    defaultValue="software-catalog"
  />
  ...
</Paper>
```

> Check out the documentation around [integrating search into plugins](../../plugins/integrating-search-into-plugins.md#create-a-collator) for how to create your own collator.

## How to customize fields in the Software Catalog index

Sometimes you will might want to have ability to control
which data passes to search index in catalog collator, or to customize data for specific kind.
You can easily do that by passing `entityTransformer` callback to `DefaultCatalogCollatorFactory`.
You can either just simply amend default behaviour, or even to write completely new document
(which should follow some required basic structure though).

> `authorization` and `location` cannot be modified via a `entityTransformer`, `location` can be modified only through `locationTemplate`.

```diff
// packages/backend/src/plugins/search.ts

const entityTransformer: CatalogCollatorEntityTransformer = (entity: Entity) => {
  if (entity.kind === 'SomeKind') {
    return {
      // customize here output for 'SomeKind' kind
    };
  }

  return {
    // and customize default output
    ...defaultCatalogCollatorEntityTransformer(entity),
    text: 'my super cool text',
  };
};

indexBuilder.addCollator({
  collator: DefaultCatalogCollatorFactory.fromConfig(env.config, {
    discovery: env.discovery,
    tokenManager: env.tokenManager,
+   entityTransformer,
  }),
});
```

## How to limit what can be searched in the Software Catalog

The Software Catalog includes a wealth of information about the components,
systems, groups, users, and other aspects of your software ecosystem. However,
you may not always want _every_ aspect to appear when a user searches the
catalog. Examples include:

- Entities of kind `Location`, which are often not useful to Backstage users.
- Entities of kind `User` or `Group`, if you'd prefer that users and groups be
  exposed to search in a different way (or not at all).

It's possible to write your own [Collator](./concepts.md#collators) to control
exactly what's available to search, (or a [Decorator](./concepts.md#decorators)
to filter things out here and there), but the `DefaultCatalogCollator` that's
provided by `@backstage/plugin-catalog-backend` offers some configuration too!

```diff
// packages/backend/src/plugins/search.ts

indexBuilder.addCollator({
  defaultRefreshIntervalSeconds: 600,
  collator: DefaultCatalogCollator.fromConfig(env.config, {
    discovery: env.discovery,
    tokenManager: env.tokenManager,
+   filter: {
+      kind: ['API', 'Component', 'Domain', 'Group', 'System', 'User'],
+   },
  }),
});
```

As shown above, you can add a catalog entity filter to narrow down what catalog
entities are indexed by the search engine.

## How to customize document type in the Software Catalog index

In some cases you might want to have the ability to change the document type in which catalog entities will be indexed by catalog collator.
Such option gives a possibility to customize SearchPage results and filters depending on which document type you would like to see results for.

You can achieve that by passing additional parameter `documentType` to the `DefaultCatalogCollatorFactory`.

Let's say that you want to have two different document types for some entities.
Our example will cover a use case in which we want to:

- Store entities of kind `User` or `Group` under document type `yourCustomDocumentType`,
- Store rest of entities under default document type `software-catalog`

To achieve that you will have to remove `User` and `Group` from your previous collator `filter` and register new `DefaultCatalogCollatorFactory` with new parameter `documentType`.

```diff
// packages/backend/src/plugins/search.ts

  indexBuilder.addCollator({
    schedule,
    factory: DefaultCatalogCollatorFactory.fromConfig(env.config, {
      discovery: env.discovery,
      tokenManager: env.tokenManager,
      filter: {
-        kind: ['API', 'Component', 'Domain', 'Resource', 'System', 'Template', 'User', 'Group'],
+        kind: ['API', 'Component', 'Domain', 'Resource', 'System', 'Template'],
      },
    }),
  });

+  indexBuilder.addCollator({
+    schedule,
+    factory: DefaultCatalogCollatorFactory.fromConfig(env.config, {
+      discovery: env.discovery,
+      tokenManager: env.tokenManager,
+      filter: {
+        kind: ['User', 'Group'],
+      },
+      documentType: 'yourCustomDocumentType',
+    }),
+  });
```

## How to customize search results highlighting styling

The default highlighting styling for matched terms in search results is your
browsers default styles for the `<mark>` HTML tag. If you want to customize
how highlighted terms look you can follow Backstage's guide on how to
[Customize the look-and-feel of your App](https://backstage.io/docs/getting-started/app-custom-theme)
to create an override with your preferred styling.

For example, the following will result in highlighted terms to be bold & underlined:

```jsx
const highlightOverride = {
  BackstageHighlightedSearchResultText: {
    highlight: {
      color: 'inherit',
      backgroundColor: 'inherit',
      fontWeight: 'bold',
      textDecoration: 'underline',
    },
  },
};
```

[obj-mode]: https://nodejs.org/dist/latest-v16.x/docs/api/stream.html#stream_object_mode
[read-stream]: https://nodejs.org/dist/latest-v16.x/docs/api/stream.html#readable-streams
[async-gen]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of#iterating_over_async_generators

## How to render search results using extensions

Extensions for search results let you customize components used to render search result items, It is possible to provide your own search result item extensions or use the ones provided by plugin packages:

### 1. Providing an extension in your plugin package

Using the example below, you can provide an extension to be used as a default result item:

```tsx
// plugins/your-plugin/src/plugin.ts
import { createPlugin } from '@backstage/core-plugin-api';
import { createSearchResultListItemExtension } from '@backstage/plugin-search-react';

const plugin = createPlugin({ id: 'YOUR_PLUGIN_ID' });

export const YourSearchResultListItemExtension = plugin.provide(
  createSearchResultListItemExtension({
    name: 'YourSearchResultListItem',
    component: () =>
      import('./components').then(m => m.YourSearchResultListItem),
  }),
);
```

If your list item accept props, you can extend the `SearchResultListItemExtensionProps` with your component specific props:

```tsx
export const YourSearchResultListItemExtension: (
  props: SearchResultListItemExtensionProps<YourSearchResultListItemProps>,
) => JSX.Element | null = plugin.provide(
  createSearchResultListItemExtension({
    name: 'YourSearchResultListItem',
    component: () =>
      import('./components').then(m => m.YourSearchResultListItem),
  }),
);
```

Additionally, you can define a predicate function that receives a result and returns whether your extension should be used to render it or not:

```tsx
// plugins/your-plugin/src/plugin.ts
import { createPlugin } from '@backstage/core-plugin-api';
import { createSearchResultListItemExtension } from '@backstage/plugin-search-react';

const plugin = createPlugin({ id: 'YOUR_PLUGIN_ID' });

export const YourSearchResultListItemExtension = plugin.provide(
  createSearchResultListItemExtension({
    name: 'YourSearchResultListItem',
    component: () =>
      import('./components').then(m => m.YourSearchResultListItem),
    // Only results matching your type will be rendered by this extension
    predicate: result => result.type === 'YOUR_RESULT_TYPE',
  }),
);
```

Remember to export your new extension:

```tsx
// plugins/your-plugin/src/index.ts
export { YourSearchResultListItem } from './plugin.ts';
```

For more details, see the [createSearchResultListItemExtension](https://backstage.io/docs/reference/plugin-search-react.createsearchresultlistitemextension) API reference.

### 2. Using an extension in your Backstage app

Now that you know how a search result item is provided, let's finally see how they can be used, for example, to compose a page in your application:

```tsx
// packages/app/src/components/searchPage.tsx
import React from 'react';

import { Grid, Paper } from '@material-ui/core';
import BuildIcon from '@material-ui/icons/Build';

import {
  Page,
  Header,
  Content,
  DocsIcon,
  CatalogIcon,
} from '@backstage/core-components';
import { SearchBar, SearchResult } from '@backstage/plugin-search-react';

// Your search result item extension
import { YourSearchResultListItem } from '@backstage/your-plugin';

// Extensions provided by other plugin developers
import { ToolSearchResultListItem } from '@backstage/plugin-explore';
import { TechDocsSearchResultListItem } from '@backstage/plugin-techdocs';
import { CatalogSearchResultListItem } from '@internal/plugin-catalog-customized';

// This example omits other components, like filter and pagination
const SearchPage = () => (
  <Page themeId="home">
    <Header title="Search" />
    <Content>
      <Grid container direction="row">
        <Grid item xs={12}>
          <Paper>
            <SearchBar />
          </Paper>
        </Grid>
        <Grid item xs={12}>
          <SearchResult>
            <YourSearchResultListItem />
            <CatalogSearchResultListItem icon={<CatalogIcon />} />
            <TechDocsSearchResultListItem icon={<DocsIcon />} />
            <ToolSearchResultListItem icon={<BuildIcon />} />
          </SearchResult>
        </Grid>
      </Grid>
    </Content>
  </Page>
);

export const searchPage = <SearchPage />;
```

> **Important**: A default result item extension should be placed as the last child, so it can be used only when no other extensions match the result being rendered. If a non-default extension is specified, the `DefaultResultListItem` component will be used.

As another example, here's a search modal that renders results with extensions:

```tsx
// packages/app/src/components/searchModal.tsx
import React from 'react';

import { DialogContent, DialogTitle, Paper } from '@material-ui/core';
import BuildIcon from '@material-ui/icons/Build';

import { DocsIcon, CatalogIcon } from '@backstage/core-components';
import { SearchBar, SearchResult } from '@backstage/plugin-search-react';

// Your search result item extension
import { YourSearchResultListItem } from '@backstage/your-plugin';

// Extensions provided by other plugin developers
import { ToolSearchResultListItem } from '@backstage/plugin-explore';
import { TechDocsSearchResultListItem } from '@backstage/plugin-techdocs';
import { CatalogSearchResultListItem } from '@internal/plugin-catalog-customized';

export const SearchModal = ({ toggleModal }: { toggleModal: () => void }) => (
  <>
    <DialogTitle>
      <Paper>
        <SearchBar />
      </Paper>
    </DialogTitle>
    <DialogContent>
      <SearchResult onClick={toggleModal}>
        <CatalogSearchResultListItem icon={<CatalogIcon />} />
        <TechDocsSearchResultListItem icon={<DocsIcon />} />
        <ToolSearchResultListItem icon={<BuildIcon />} />
        {/* As a "default" extension, it does not define a predicate function, 
        so it must be the last child to render results that do not match the above extensions */}
        <YourSearchResultListItem />
      </SearchResult>
    </DialogContent>
  </>
);
```

There are other more specific search results layout components that also accept result item extensions, check their documentation: [SearchResultList](https://backstage.io/storybook/?path=/story/plugins-search-searchresultlist--with-result-item-extensions) and [SearchResultGroup](https://backstage.io/storybook/?path=/story/plugins-search-searchresultgroup--with-result-item-extensions).
