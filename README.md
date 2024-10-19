![monaco with lsp](https://media.dev.to/dynamic/image/width=1000,height=420,fit=cover,gravity=auto,format=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fastbge8jidzrv7c6jex2.PNG)
# So, you want to set up a Monaco editor with a language server
Currently it's not simple to connect an LSP language server to a custom editor (not Neovim and VSCode), the docs are usually sparse and there is lack of simple and documented projects that implement that.
I faced this, and decided to dive down and document my way, so I hope this post helps anyone.

In this post I will cover:
- Basics of monaco-editor
- Using monaco-editor with several editor windows
- Using monaco-vscode-api package and setting up the basic language features
- Adding monaco-languageclient and Python LSP

I tried to document all the sources along the way, so you can go and learn more where needed.

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Simple monaco-editor](#simple-monaco-editor)
   * [Initial project](#initial-project)
   * [Adding workers](#adding-workers)
   * [Adding languages](#adding-languages)
   * [Adding more editors](#adding-more-editors)
- [VSCode-API](#vscode-api)
   * [New beginning](#new-beginning)
   * [Adding language](#adding-language)
- [Introducing language server](#introducing-language-server)
- [Where to go next](#where-to-go-next)

<!-- TOC end -->

<!-- TOC --><a name="simple-monaco-editor"></a>
## Simple monaco-editor
Let's start with the most simple Monaco setup. I will use vanilla TS with Vite and Bun as package manager, so I hope it will be simple to extrapolate into different frameworks. 
You can also find [similar example](https://github.com/microsoft/monaco-editor/tree/main/samples/browser-esm-vite-react) written with React in the official repo.

**Feel free to skip this part if you already know about basic monaco-editor setup.**

<!-- TOC --><a name="initial-project"></a>
### Initial project

First, let's init a Vite project as described in [Bun docs](https://bun.sh/guides/ecosystem/vite) and install `monaco-editor`:

```shell
bun create vite my-monaco-editor
cd my-monaco-editor
bun install
bun add monaco-editor
```

Then, all we need is a simple html page with one `div` -- it will be the editor.

```html
<!-- index.html -->
<html>
  <body>
    <div id="editor"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

Add little style:

```css
/* style.css */
body {
  background-color: #242424;
}

#editor {
  margin: 10vh auto;
  width: 720px;
  height: 20vh;
}
```

And now we can create a Monaco editor instance, which will automatically fill the div with interactive editor:

```typescript
// main.ts
import './style.css'
import * as monaco from 'monaco-editor';

monaco.editor.create(document.getElementById('editor')!, {
	value: "Hello world!",
});
```

Voila! Working Monaco editor is here.

![Basic Monaco setup](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/08xwuqpydvsjgkg9kur1.png)

<!-- TOC --><a name="adding-workers"></a>
### Adding workers
If you look into the DevTools console, you may notice a warning:

> Could not create web worker(s). Falling back to loading web worker code in main thread, which might cause UI freezes. Please see https://github.com/microsoft/monaco-editor#faq


That's because Monaco editor usually separates text processing and UI interaction into different processes, so they work asynchronously without interfering with each other.

One can do it manually, like this:

```typescript
// main.ts
import './style.css'
import * as monaco from 'monaco-editor';
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker';

window.MonacoEnvironment = {
	getWorker(_workerId: any, _label: string) {
		return new editorWorker();
	}
};
monaco.editor.create(document.getElementById('editor')!, {
	value: "Hello world!",
});
```

We can provide workers for text processing using `window.MonacoEnvironment` attribute. The getWorker function receives `label` - which is the name of a worker required by editor. Since currently we do not use any languages, the default `editorWorker` will do. 

Now everything works and there is no any warnings in the console.

<!-- TOC --><a name="adding-languages"></a>
### Adding languages

Now, since Monaco is code editor, let's add some coding language processing. This can be done by adding `language` attribute to the options object in `monaco.editor.create` call:

```typescript
// main.ts
monaco.editor.create(document.getElementById('editor')!, {
	value: "console.log('Hello world!');",
	language: "typescript"
});
```

However, we did not provide the corresponding worker yet. Hopefully, Monaco provides a built-in worker for typescript and javascript:

```typescript
// main.ts
import './style.css'
import * as monaco from 'monaco-editor';
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker';
import tsWorker from 'monaco-editor/esm/vs/language/typescript/ts.worker?worker';

window.MonacoEnvironment = {
	getWorker(_workerId: any, label: string) {
		if (label === 'typescript' || label === 'javascript') {
			return new tsWorker();
		}
		return new editorWorker();
	}
};
monaco.editor.create(document.getElementById('editor')!, {
	value: "console.log('Hello world!');",
	language: "typescript"
});
```
We need first to check, which worker is requested, and then return the corresponding one. When we create an editor with given `language`, the Monaco calls getWorker, providing `language` as `label` parameter. 

However, this is true only [for some subset of languages](https://stackoverflow.com/questions/57921084/where-is-the-monaco-editor-autocompletion-located?rq=3), which are built into Monaco by default:
- json
- css
- html
- typescript
- javascript

For other languages Monaco provides less features out of the box and uses the default `editorWorker`. So if your editor is only for Python, you can leave just `editorWorker` in the `getWorkter` function, but still, provide `language: "python"` when creating an editor:

```typescript
// main.ts
import './style.css'
import * as monaco from 'monaco-editor';
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker';

window.MonacoEnvironment = {
	getWorker(_workerId: any, _label: string) {
		return new editorWorker();
	}
};
monaco.editor.create(document.getElementById('editor')!, {
	value: "print('Hello world!')",
	language: "python"
});
```

<!-- TOC --><a name="adding-more-editors"></a>
### Adding more editors
Imagine that you need more than one editor, e.g. for different files. The most straightforward path is to just create another `div` and call `monaco.editor.create` one more time:

```html
<!-- index.html -->
<html>
  <body>
    <div id="editor1"></div>
    <div id="editor2"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>

```
```css
/* style.css */
body {
  background-color: #242424;
}

#editor1, #editor2 {
  margin: 10vh auto;
  width: 720px;
  height: 20vh;
}

```
```typescript
// main.ts
import './style.css'
import * as monaco from 'monaco-editor';
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker';

window.MonacoEnvironment = {
	getWorker(_workerId: any, label: string) {
		return new editorWorker();
	}
};
monaco.editor.create(document.getElementById('editor1')!, {
	value: "print('Hello world 1!')",
	language: "python"
});

monaco.editor.create(document.getElementById('editor2')!, {
	value: "print('Hello world 2!')",
	language: "python"
});
```

This will work, but not ideally -- Monaco will try to autocomplete variable names from different editor:
![Monaco autocomplete from different editor](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ygplt4fo36amjlefzts9.png)
It's easy to fix, just add `wordBasedSuggestions` field set to `currentDocument`:

```typescript
monaco.editor.create(document.getElementById('editor2')!, {
	value: "print('Hello world 2!')",
	language: "python",
	wordBasedSuggestions: 'currentDocument'
});
```


That was the basic possible setup of `monaco-editor`. If you are using it for TypeScript/CSS/HTML, that may be enough because of the built-in workers. However, if you need it for Python or any other language, you may need to integrate a custom LSP to support advanced features like IntelliSense, default keyword autocompletion, code navigation, linting, etc.
<!-- TOC --><a name="vscode-api"></a>
## VSCode-API
Let's try to build a Monaco editor with full LSP functionality for Python.

Unfortunately, support for LSP is not built in natively to Monaco, so you can't just do something like:
```typescript
// main.ts
import './style.css'
import * as monaco from 'monaco-editor';
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker';
import languageClient from 'monaco-lsp' // doesn't exist

window.MonacoEnvironment = {
	getWorker(_workerId: any, _label: string) {
		return new editorWorker();
	}
};

const lsp = new languageClient( // doesn't exist
    serverUri="ws://localhost:5007", 
    rootUri="file:///", 
    languageId="python"
)
monaco.editor.create(document.getElementById('editor')!, {
	value: "print('Hello world!')",
	language: "python",
	lsp: lsp // doesn't work
});

```

However, most of the language servers work with VSCode out of the box via extensions. And since VSCode is built around Monaco, it's possible to integrate VSCode API (e.g. extensions and other stuff) into Monaco. Including LSP support.

Package [monaco-vscode-api](https://github.com/CodinGame/monaco-vscode-api) does exactly that. But moreover, it in some way redesigns the `monaco-editor`, making it much more modular (but also more complex).

The documentation on it is not very good (some info in the README and examples in issues and the demo). So, mostly I've figured it all out via trial and error and 3 issues focused on struggle of implementing the basic functionality:
- [# Minimal example #444](https://github.com/CodinGame/monaco-vscode-api/issues/444)
- [# Minimal example with extension support #465](https://github.com/CodinGame/monaco-vscode-api/issues/465)
- [# I have officially spent four hours of my lifespan to set a color theme. #510](https://github.com/CodinGame/monaco-vscode-api/issues/510)


It changes the code drastically, but some main concepts are the same. Let's start again with a minimal example to demonstrate that.

<!-- TOC --><a name="new-beginning"></a>
### New beginning

Let's create a new project in the same way as before, but instead of `monaco-editor` we will install `monaco-vscode-api` packages
```shell
# before proceeding make sure you are not in an existing project
bun create vite my-monaco-api-editor
cd my-monaco-api-editor
bun install

bun add vscode@npm:@codingame/monaco-vscode-api
bun add monaco-editor@npm:@codingame/monaco-vscode-editor-api
bun add -D @types/vscode
```
These packages names may look weird, because they use aliases to be used as enhanced drop-in replacement for `vscode` and `monaco-editor` packages. They change their functionality to allow the usage of VSCode services and extensions in Monaco, but provide the same interface as the original packages.

Then, again, all we need is a simple html page with one `div` -- it will be the editor.

```html
<!-- index.html -->
<html>
  <body>
    <div id="editor"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

Add little style:

```css
/* style.css */
body {
  background-color: #242424;
}

#editor {
  margin: 10vh auto;
  width: 720px;
  height: 20vh;
}
```

As for `main.ts`, we can start with totally same example, since we've just used drop-in replacement:

```typescript
// main.ts
import './style.css'
import * as monaco from 'monaco-editor';

import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker';

window.MonacoEnvironment = {
  getWorker: function (_moduleId, _label) {
	return new editorWorker();
  }
}
monaco.editor.create(document.getElementById('editor')!, {
	value: "Hello world!",
});

```

In the `monaco-vscode-api` repo's issues and demo project you will meet the following variant of adding workers:

```typescript
// main.ts
import './style.css'
import * as monaco from 'monaco-editor';

export type WorkerLoader = () => Worker;
const workerLoaders: Partial<Record<string, WorkerLoader>> = {
	TextEditorWorker: () => new Worker(new URL('monaco-editor/esm/vs/editor/editor.worker.js', import.meta.url), { type: 'module' })
	})
}
window.MonacoEnvironment = {
  getWorker: function (_workerId, label) {
	const workerFactory = workerLoaders[label]
    if (workerFactory != null) {
      return workerFactory()
    }
	throw new Error(`Worker ${label} not found`)
  }
}

monaco.editor.create(document.getElementById('editor')!, {
	value: "Hello world!",
});

```
It's basically the same, but more strictly checks if the required worker is implemented. Also, the worker initialization is a little different but still functionally same - `TextEditorWorker` is label for the default `editorWorker` from the previous examples.

There are some nuances to account for in your bundler. They are described in the [Troubleshooting section](https://github.com/CodinGame/monaco-vscode-api?tab=readme-ov-file#troubleshooting) of the repo. Since I'm using Vite here, I'll provide details for Vite users below.

{% details For Vite users %}
It uses `import.meta.url` base which doesn't work well with Vite out of the box. So if you are using Vite, add this to your `vite.config.ts` (create it if not yet):
 ```typescript
 import type { UserConfig } from 'vite'
 import importMetaUrlPlugin from '@codingame/esbuild-import-meta-url-plugin'
 
 export default {
    optimizeDeps: {
        esbuildOptions: {
          plugins: [importMetaUrlPlugin]
        }
      }
} satisfies UserConfig
```
And install the corresponding package
```shell
bun add @codingame/esbuild-import-meta-url-plugin
```
{% enddetails %}

Further, I will use the latter approach to worker initialization so the reader is more used to the notation usually met in the repo. Also, we will know if any worker is not added properly via error in the console.
<!-- TOC --><a name="adding-language"></a>
### Adding language
So, we've built a basic text editor using new `monaco-vscode-api` as a drop-in replacement for `monaco-editor`. Let's try to add Python highlighting. Previously, it was made by adding `language` attribute to the `monaco.editor.create` options object.

However, if we add `language`,  nothing changes and even the highlighting is absent:

```typescript
import './style.css'
import * as monaco from 'monaco-editor';

import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker';

export type WorkerLoader = () => Worker;
const workerLoaders: Partial<Record<string, WorkerLoader>> = {
	TextEditorWorker: () => new Worker(new URL('monaco-editor/esm/vs/editor/editor.worker.js', import.meta.url), { type: 'module' })
	})
}
window.MonacoEnvironment = {
  getWorker: function (_workerId, label) {
	const workerFactory = workerLoaders[label]
    if (workerFactory != null) {
      return workerFactory()
    }
	throw new Error(`Worker ${label} not found`)
  }
}

monaco.editor.create(document.getElementById('editor')!, {
	value: "print('Hello world!')",
	language: "python"
});

```
![Monaco with no highlighting](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nldo5ix9jqetcqdc7uab.png)
That's because the default workers which supplied most of the functions doesn't work in `monaco-vscode-api` the same way they did in `monaco-editor`. Now, the most of the editor functionality is based on **VSCode services** - components which provide specific functions, even the basic ones. Here is [a large list of services](https://github.com/CodinGame/monaco-vscode-api?tab=readme-ov-file#monaco-standalone-services), supported by `monaco-vscode-api`. And to make this functionality work, one need to add appropriate services manually.

E.g. to support highlighting, now we need to add the following services:

- **Textmate**: `@codingame/monaco-vscode-textmate-service-override`
	- Allows to use textmate grammars to tokenize languages for highlighting.
- **Themes**: `@codingame/monaco-vscode-theme-service-override`
    - Allows to use VSCode themes. Note: original Monaco themes are different from VSCode themes in this package. VSCode themes are represented and installed as VSCode extensions. Also, see [# I have officially spent four hours of my lifespan to set a color theme. #510](https://github.com/CodinGame/monaco-vscode-api/issues/510) issue for details.
- **Languages**: `@codingame/monaco-vscode-languages-service-override`
    - Allows to account for the `language` field and setup textmate grammars for highlighting and other language specific functions.
And import the corresponding language and theme extensions (see below).


Adding services is simple, install the corresponding package from the [list](https://github.com/CodinGame/monaco-vscode-api?tab=readme-ov-file#monaco-standalone-services) and pass the  `...get*ServiceOverride()` into `initialize` function from `vscode/services` before creating editors.
Let's try this.

Installing packages for services and extensions:
```shell
# splitted into several commands for readibility
bun add @codingame/monaco-vscode-textmate-service-override 
bun add @codingame/monaco-vscode-theme-service-override 
bun add @codingame/monaco-vscode-languages-service-override
bun add @codingame/monaco-vscode-python-default-extension
bun add @codingame/monaco-vscode-theme-defaults-default-extension
```


Adding services to the editor:
```typescript
// main.ts
import './style.css'
import * as monaco from 'monaco-editor';

// importing installed services
import { initialize } from 'vscode/services'
import getLanguagesServiceOverride from "@codingame/monaco-vscode-languages-service-override";
import getThemeServiceOverride from "@codingame/monaco-vscode-theme-service-override";
import getTextMateServiceOverride from "@codingame/monaco-vscode-textmate-service-override";

// adding worker
export type WorkerLoader = () => Worker;
const workerLoaders: Partial<Record<string, WorkerLoader>> = {
	TextEditorWorker: () => new Worker(new URL('monaco-editor/esm/vs/editor/editor.worker.js', import.meta.url), { type: 'module' })
	})
}
window.MonacoEnvironment = {
  getWorker: function (_workerId, label) {
	const workerFactory = workerLoaders[label]
    if (workerFactory != null) {
      return workerFactory()
    }
	throw new Error(`Worker ${label} not found`)
  }
}

// adding services
await initialize({
    ...getTextMateServiceOverride(),
    ...getThemeServiceOverride(),
    ...getLanguagesServiceOverride(),
});

monaco.editor.create(document.getElementById('editor')!, {
	value: "print('Hello world!')",
	language: "python"
});
```

This will work without any warnings or errors, but the highlighting is not there yet. To add it, we need to finally integrate 
1) Python language default extension which will provide Python grammar; 
2) TextMate worker which will tokenize the code based on the grammar; 
3) Theme, so different keywords can have unique color.

Python language and theme are both VSCode extensions. Installation of extensions [described in detail in README.md](https://github.com/CodinGame/monaco-vscode-api?tab=readme-ov-file#vscode-api-usage). Fortunately, for default VSCode extensions like the ones we need, there are prebuilt packages by the repo's authors:
- `@codingame/monaco-vscode-python-default-extension` for Python
- `@codingame/monaco-vscode-theme-defaults-default-extension` for default VSCode themes
Adding them to the project is as simple as installing the packages and adding the corresponding imports to the beginning of the `main.ts`.
```typescript
// main.ts
import '@codingame/monaco-vscode-python-default-extension';
import "@codingame/monaco-vscode-theme-defaults-default-extension";

... // rest of the code
```

To integrate the TextMate worker, we need to add it to the `workerLoaders` map:
```typescript
// main.ts
...

const workerLoaders: Partial<Record<string, WorkerLoader>> = {
	TextEditorWorker: () => new Worker(new URL('monaco-editor/esm/vs/editor/editor.worker.js', import.meta.url), { type: 'module' }),
	TextMateWorker: () => new Worker(new URL('monaco-editor/esm/vs/language/textmate/textmate.worker.js', import.meta.url), { type: 'module' })
}
...
```

{% details How to find necessary services and/or extensions? %}
You can find a full list of services and extensions here: https://www.npmjs.com/search?q=%40codingame%2Fmonaco-vscode-*-default-extension
There is no full documentation so, to find out what you need you usually look into issues/demo/other projects using `monaco-vscode-api`, and copy that, or intuitively add services/extensions based on their name until it is not working. At least, I've not found better way yet.
{% enddetails %}
And voila, the final code with Python highlighting support:
```typescript
import '@codingame/monaco-vscode-python-default-extension';
import "@codingame/monaco-vscode-theme-defaults-default-extension";

import './style.css'
import * as monaco from 'monaco-editor';
import { initialize } from 'vscode/services'

 
import getLanguagesServiceOverride from "@codingame/monaco-vscode-languages-service-override";
import getThemeServiceOverride from "@codingame/monaco-vscode-theme-service-override";
import getTextMateServiceOverride from "@codingame/monaco-vscode-textmate-service-override";

export type WorkerLoader = () => Worker;
const workerLoaders: Partial<Record<string, WorkerLoader>> = {
	TextEditorWorker: () => new Worker(new URL('monaco-editor/esm/vs/editor/editor.worker.js', import.meta.url), { type: 'module' }),
	TextMateWorker: () => new Worker(new URL('monaco-editor/esm/vs/language/textmate/textmate.worker.js', import.meta.url), { type: 'module' })
}

window.MonacoEnvironment = {
  getWorker: function (_moduleId, label) {
	console.log('getWorker', _moduleId, label);
	const workerFactory = workerLoaders[label]
    if (workerFactory != null) {
      return workerFactory()
    }
	throw new Error(`Worker ${label} not found`)
  }
}

await initialize({
	...getTextMateServiceOverride(),
	...getThemeServiceOverride(),
	...getLanguagesServiceOverride(),
});

monaco.editor.create(document.getElementById('editor')!, {
	value: "import numpy as np\nprint('Hello world!')",
	language: 'python'
});
```

Note: afaik, to turn the highlighting on you may need to manually edit your code (e.g. add and remove a whitespace), so the TextMate worker starts to work. I may update this later if I find a robust solution.


<!-- TOC --><a name="introducing-language-server"></a>
## Introducing language server
Now, let's finally add language server. For this one, we will need to use a cousin package of `monaco-vscode-api` called `monaco-languageclient` which actively utilizes the former.

We also will need a language server itself. 

{% details Note on LSP servers %}
Usually when using VSCode, you just select a language and install the corresponding language server extension from Marketplace, e.g. Pyright of Ruff for Python. Under the hood, most of these VSCode language server extensions utilize `vscode-languageclient` [api](https://github.com/microsoft/vscode-languageserver-node/blob/a561f1342ba94ad7f550cb15446f65432f5e1367/client/src/node/main.ts#L128). The API allows to launch LSP server in several ways, e.g. as a node module running in runtime provided by VSCode itself, or as a child process via runnable command.
You can take a look at the [Pylyzer](https://github.com/mtshiba/pylyzer/blob/main/extension/src/extension.ts) Python LSP extension to see an example of usage of the API.

**Note that in order to use it, you need a runtime that has access to your files.**

There is a possibility to [add VSCode server to your Monaco project](https://github.com/CodinGame/monaco-vscode-api/blob/main/docs/vscode_server.md) and use it to launch language servers, however it adds additional complexity and dependency. 
In this guide I will avoid it.

There are other ways to run Language Server, e.g. one can create a new language server or a wrapper for existing one with [`pygls`](https://pygls.readthedocs.io/en/latest/), to run it as Python process providing websocket server. Here is a [great guide](https://tomassetti.me/integrating-textx-and-monaco-a-non-tutorial/) with introduction to language servers and monaco language client. Other similar option but for Rust is [`tower-lsp`](https://github.com/ebkalderon/tower-lsp).
{% enddetails %}
Let's go with the most simple way for Python -- use [`python-lsp-server`](https://github.com/python-lsp/python-lsp-server/tree/develop), which provides websocket LSP server with all bells and whistles out of the box.  

The following example will be based on a [bare client example implementation](https://github.com/TypeFox/monaco-languageclient/blob/main/packages/examples/src/bare/client.ts) from the languageclient repo.

To proceed, we will need to install two additional packages:
```shell
bun add vscode-ws-jsonrpc
bun add monaco-languageclient
```

Then, let's create a file `lsp-client.ts`. Here we will write initialization functions for the LSP client. There we will handle WebSocket connection with the server.

```typescript
// lsp-client.ts
import { WebSocketMessageReader } from 'vscode-ws-jsonrpc';
import { CloseAction, ErrorAction, MessageTransports } from 'vscode-languageclient/browser.js';
import { WebSocketMessageWriter } from 'vscode-ws-jsonrpc';
import { toSocket } from 'vscode-ws-jsonrpc';
import { MonacoLanguageClient } from 'monaco-languageclient';

export const initWebSocketAndStartClient = (url: string): WebSocket => {
    const webSocket = new WebSocket(url);
    webSocket.onopen = () => {
	    // creating messageTransport
        const socket = toSocket(webSocket);
        const reader = new WebSocketMessageReader(socket);
        const writer = new WebSocketMessageWriter(socket);
        // creating language client
        const languageClient = createLanguageClient({
            reader,
            writer
        });
        languageClient.start();
        reader.onClose(() => languageClient.stop());
    };
    return webSocket;
};
const createLanguageClient = (messageTransports: MessageTransports): MonacoLanguageClient => {
    return new MonacoLanguageClient({
        name: 'Sample Language Client',
        clientOptions: {
            // use a language id as a document selector
            documentSelector: ['python'],
            // disable the default error handler
            errorHandler: {
                error: () => ({ action: ErrorAction.Continue }),
                closed: () => ({ action: CloseAction.DoNotRestart })
            }
        },
        // create a language client connection from the JSON RPC connection on demand
        connectionProvider: {
            get: async (_encoding: string) => messageTransports
        }
    });
};
```
The key concept here is `messageTransports` parameter in `createLanguageClient` function. It is a pair of initialized websocket reader and writer that allow to communicate with the server.

Now, all we need to make it work is to run the `initWebSocketAndStartClient` function from `main.ts` providing url and port of the websocket language server:

```typescript
import '@codingame/monaco-vscode-python-default-extension';
import "@codingame/monaco-vscode-theme-defaults-default-extension";

import './style.css'
import * as monaco from 'monaco-editor';
import { initialize } from 'vscode/services'

// we need to import this so monaco-languageclient can use vscode-api
import "vscode/localExtensionHost";
import { initWebSocketAndStartClient } from 'lsp-client'

// everything else is the same except the last line
import getLanguagesServiceOverride from "@codingame/monaco-vscode-languages-service-override";
import getThemeServiceOverride from "@codingame/monaco-vscode-theme-service-override";
import getTextMateServiceOverride from "@codingame/monaco-vscode-textmate-service-override";

export type WorkerLoader = () => Worker;
const workerLoaders: Partial<Record<string, WorkerLoader>> = {
	TextEditorWorker: () => new Worker(new URL('monaco-editor/esm/vs/editor/editor.worker.js', import.meta.url), { type: 'module' }),
	TextMateWorker: () => new Worker(new URL('monaco-editor/esm/vs/language/textmate/textmate.worker.js', import.meta.url), { type: 'module' })
}

window.MonacoEnvironment = {
  getWorker: function (_moduleId, label) {
	console.log('getWorker', _moduleId, label);
	const workerFactory = workerLoaders[label]
    if (workerFactory != null) {
      return workerFactory()
    }
	throw new Error(`Worker ${label} not found`)
  }
}

await initialize({
	...getTextMateServiceOverride(),
	...getThemeServiceOverride(),
	...getLanguagesServiceOverride(),
});

monaco.editor.create(document.getElementById('editor')!, {
	value: "import numpy as np\nprint('Hello world!')",
	language: 'python'
});

// start websocket lsp client on port 5007 
// (you can choose any port, just make sure the server uses the same)
initWebSocketAndStartClient("ws://localhost:5007/")
```

Now, just install and run `python-lsp-server` on the port you selected:
```shell
pip install python-lsp-server
pylsp --ws --port 5007
```

And here we go:

![Monaco with LSP](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6ldgm4hj1v2o9ey15hoo.gif)


<!-- TOC --><a name="where-to-go-next"></a>
## Where to go next
- You can reduce the amount of boilerplate (e.g. adding services for basic functionality like themes, highlighting, etc.) by using [`monaco-editor-wrapper`](https://www.npmjs.com/package/monaco-editor-wrapper)
- You can dive deeper into the concept of [models](https://www.npmjs.com/package/monaco-editor-wrapper) in order to better control LSP features between files in your project
- You can try to set up some nodejs-based language server like pyright or basedpyright. 
- Look at the [examples](https://github.com/TypeFox/monaco-languageclient/tree/main?tab=readme-ov-file#examples-overview) in `monaco-languageclient` to learn more.
