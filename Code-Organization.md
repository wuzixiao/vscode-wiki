Code consists of a layered and modular `core` that can be extended using extensions. Extensions are run in a separate process refered to as the
`extension host.` Extensions are implemented by utilizing the [extension API](https://code.visualstudio.com/docs/extensions/overview).

# Layers

The `core` is partitioned into the following layers:
- `base`: Provides general utilities and user interface building blocks
- `platform`: Defines service injection support and the base services for Code
- `editor`: The "Monaco" editor is available as a separate downloadable component
- `languages`: For historical reasons, not all languages are implemented as extensions (yet) - as Code evolves we will migrate more languages to towards extensions
- `workbench`: Hosts the "Monaco" editor and provides the framework for "viewlets" like the Explorer, Status Bar, or Menu Bar, leveraging [Electron](http://electron.atom.io/) to implement the Code desktop application.

# Target Environments
The `core` of Code is fully implemented in [TypeScript](https://github.com/microsoft/typescript). Inside each layer the code is organized by the target runtime environment. This ensures that only the runtime specific APIs are used. In the code we distinguish between the following target environments:
- `common`: Source code that only requires basic JavaScript APIs and run in all the other target environments
- `browser`: Source code that requires the `browser` APIs like access to the DOM
  - may use code from: `common`
- `node`: Source code that requires [`nodejs`](https://nodejs.org) APIs
  - may use code from: `common`
- `electron-browser`: Source code that requires the [Electron renderer-process](https://github.com/atom/electron/tree/master/docs#modules-for-the-renderer-process-web-page) APIs
  - may use code from: `common`, `browser`, `node`
- `electron-main`: Source code that requires the [Electron main-process](https://github.com/atom/electron/tree/master/docs#modules-for-the-main-process) APIs
  - may use code from: `common`, `node`

# Dependency Injection

The code is organised around services of which most are defined in the `platform` layer. Services get to its clients via `constructor injection`. 

A service definition is two parts: (1) the interface of a service, and (2) a service identifier - the latter is required because TypeScript doesn't use nominal but structural typing. A service identifier is a decoration (as proposed for ES7) and should have the same name as the service interface. 

Declaring a service dependency happens by adding a corresponding decoration to a constructor argument. In the snippet below `@IModelService` is the service identifier decoration and `IModelService` is the (optional) type annotation for this argument. When a dependency is optional, use the `@optional` decoration otherwise the instantiation services throws an error.

```javascript
class Client {
  constructor(
    @IModelService modelService: IModelService, 
    @optional(IEditorService) editorService: IEditorService
  ) {
    // use services
  }
}
```

Use the instantiation service to create instances for service consumers, like so `instantiationService.createInstance(Client)`. Usually, this is done for you when being registered as a contribution, like a Viewlet or Language.

# Workbench Parts

The VS Code workbench (`vs/workbench`) is composed of many things to provide a rich development experience. Examples include full text search, integrated git and debug. At its core, the workbench does not have direct dependencies to all these parts. Instead, we use an internal (as opposed to real extension API) mechanism to contribute these parts to the workbench. 

Parts that are contributed to the workbench all live inside the `vs/workbench/parts` folder. There are some rules around this folder:
- there cannot be any dependency from outside `vs/workbench/parts` into `vs/workbench/parts`
- every part should expose its internal API from a single file (e.g. `vs/workbench/parts/search/common/search.ts`)
- a part is allowed to depend on the internal API of another part (e.g. the git part may depend on  `vs/workbench/parts/search/common/search.ts`)
- a part should never reach into the internals of another part (internal is anything inside a part that is not in the single common API file)