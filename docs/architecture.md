## Reflection

todo

## Assembly

todo

## Plugins

### Diagrams

#### Plugin Tree Shaking

![plugin-tree-shaking](https://dsc.cloud/661643/plugin-tree-shaking.png)

### Glossary

#### Entrypoint

A plugin entrypoint is responsible for returning the plugin's manifest input. This is what users actually deal with at the app layer.

```ts
import { prisma } from 'nexus-plugin-prisma'
//       ~~~~~~ <------------- The entrypoint, returns a manifest input
import { use } from 'nexus'

use(prisma())
```

#### Manifest

A plugin manifest describes logistical information about a plugin like where its package file is located on disk, what versions of Nexus it is compatible with, and more.

Plugin manifests are created by Nexus by processing the manifest inputs returned by plugin entrypoints.

```ts
import { prisma } from 'nexus-plugin-prisma'

const prismaPluginManifestInput = prisma()
```

This lazy approach allows Nexus the flexibility it needs to provide features like automatic environment variable plugin settings injection and tree-shaking.

#### Manifest Input

A manifest input is the view of manifests that plugin authors work with. It models the minimum information that Nexus needs to build the plugin manifest. It is also more human friendly, for example minimizing the number of required fields. Developer experience is not the primary motivation here however. We want Nexus to control as much as possible, so the less the human has to specify the more we achieve this goal. For example we ask for a path to package json rather than the contents of it. We want Nexus to be the one that reads it, when and how it wants.

#### Dimension

A plugin dimension represents a high-level area of plugin concern that are separated by module from other dimensions. In other words, each dimension has its own entrypoint. Nexus will directly import it. A dimension is found by Nexus from Nexus reading the Plugin manifest.

There are three dimensions: worktime, testtime, and runtime. Worktime allows plugins to tap into the CLI, the builder, dev mode, and more. Testtime allows plugins to tap into features of the Nexus testing component. Runtime allows plugins to tap into the API.

Directionally Nexus is on a good track, but there is work still left to do. The names are a bit confusing when you dig into the details, and the supposed separation between worktime/runtime has undesirable "loopholes" because of reflection. Details;

1.  Runtime dimension does not mean plugging exclusively into what is run in your production code. There are actually reasons to plug into the runtime for ostensibly worktime benefit... This is due to Nexus' so-called reflection system, wherein the app is run in the background during development for development purposes.

2.  The rationale for splitting worktime from runtime is clear, tree-shaking alone makes the case for it. However the separation between worktime and testing is less clear, perhaps nonsense, and so may be revisited in the future.

3.  We've talked about motivation for separating worktime from runtime, yet there are runtime parts for worktime (reflection). What this means is that expensive dependencies can make there way into the runtime dimension that a user should actually _not_ be paying for in production runtime.

#### Lens

A plugin lens is just a specialized api into a subset of Nexus to hook into, extend, manipulate, and react to it during execution. The name "lens" is arbitrary. The choice comes from it being "view" into Nexus. Each dimension has its own specialized lens.

#### Dimension Entrypoint

Just like the plugin has a top level entrypoint so to does each dimension within the plugin have its own entrypoint. These sub-entrypoints can be thought as sub-plugins, with the top-level plugin just being a grouping mechanism.

### Comparisons to Other Systems

#### Rollup

- Like Rollup plugins are prefixed with `<tool-name>-plugin-<plugin-name>`
- We have considered but so far not put first-party Nexus plugins under the pattern `@nexus/plugin-<plugin-name>`. Rollup made this transition retroactively.
- Rollup suggests plugins have a default export so that are much easier to use on the command line. Nexus suggests plugins also have default exports for similar system-usability reasons (not cli in Nexus' case, but other future features maybe like auto-use).

### Loading Flow

todo  
what follows is a stub

1. capture the used plugins in the app
1. validate entrypoints
1. transform entrypoints into manifests
1. for each dimension (work, test, run) in the manifest
   1. import it
   1. catch any import errors
   1. validate imported value
   1. load plugin
   1. catch any load errors

## Build Flow

1. The app layout is calculated  
   We discover things like where the entrypoint is, if any, and where [Nexus modules](/guides/project-layout?id=nexus-modules) are, if any.
1. Worktime plugins are loaded (see [Plugin Loading Flow](#plugin-loading-flow))
1. Typegen is acquired  
   This step is about processes that reflect upon the app's source code to extract type information that will be automatically used in other parts of the app. This approach is relatively novel among Node tools. There are dynamic and static processes. The static ones use the TypeScript compiler API while the dynamic ones literally run the app with node in a special reflective mode.

   Dynamic has the benefit of being able to produce types that IDE's can pick up for use in not just TypeScript but also JavaScript. It works for the schema typegen because the GraphQL Schema builders permit generating accurate derived TypeScript. Dynamic works regardless of the abstractions users through over them. On the downside, dynamic is riskier because runtime errors in the app can halt its completion. When the types to be generated are based upon arbitrary code, the task becomes one of effectively re-writing TypeScript and thus impractical.

   Static doesn't have to deal with the unpredictabilities of running an app and so has the benefit of being easier to reason about in a sense. It also has the benefit of extracting accurate type information using the native TS system whereas dynamic relies on building TS types from scratch. This makes static a fit for arbitrary code. On the downside, robust AST processing is hard work, and so, so far, static restricts how certain expressions can be written, otherwise AST traversal fails.

   1. A start module is created in memory. It imports the entrypoint and all [Nexus modules](/guides/project-layout?id=nexus-modules). It registers an extension hook to transpile the TypeScript app on the fly as it is run. The transpilation uses the project's tsconfig but overrides target and module so that it is runnable by Node (10 and up). Specifically es2015 target and commonjs module. For example if user had module of `esnext` the transpilation result would not be runnable by Node.
   1. The start module is run in a sub-process for maximum isolation. (we're looking at running within workers [#752](https://github.com/graphql-nexus/nexus/issues/752))
   1. In parallel, a TypeScript instance is created and the app source is statically analyzed to extract context types. This does not require running the app at all. TypeScript cache called tsbuildinfo is stored under `node_modules/.nexus`.

1. A new TypeScript instance is created so that the types generated in the previous step are picked up by the checker. This should be faster because it reuses the TypeScript cache created in the previous step.
1. The app is type checked
1. The app is transpiled
1. The app is emitted into `.nexus/build`. This convention keeps derived files in a well known generally ignored location.
1. A production-oriented start module is generated differing in the following ways:
   - paths are relative
   - typescript not hooked into module extensions
   - plugins are imported for tree-shaking

## Glossary

### Assembly

The process in which the app configuration (settings, schema type defs, used plugins, etc.) is processed into a runnable state. Nexus apps are only ever assembled once in their lifecycle.

### Dynamic Reflection

The process in which the app is run in order to extract data out of it. The data in turn can be used for anything. For example typegen to provide better TypeScript types or GraphQL SDL generation.

### Static Reflection

The process in which the app is analyzed with the TS API to extract data out of it. The data in turn can be used for anything. For example typegen to provide better TypeScript types.

### Reflection

The general idea of the app source code or its state at runtime being analyzed to extract data.
