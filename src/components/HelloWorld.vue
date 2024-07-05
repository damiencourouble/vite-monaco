<template>
  <div class="editor-container">
    <div id="monaco-editor-root" class="editor"></div>
  </div>
</template>

<script>
import JSZip from "jszip";
import getThemeServiceOverride from "@codingame/monaco-vscode-theme-service-override";

import getConfigurationServiceOverride from "@codingame/monaco-vscode-configuration-service-override";
import getTextmateServiceOverride from "@codingame/monaco-vscode-textmate-service-override";
import "@codingame/monaco-vscode-theme-defaults-default-extension";

import { CloseAction, ErrorAction } from "vscode-languageclient";
import "@codingame/monaco-vscode-python-default-extension";
import { useWorkerFactory } from "monaco-editor-wrapper/workerFactory";
import { MonacoEditorLanguageClientWrapper } from "monaco-editor-wrapper";
import {
  BrowserMessageReader,
  BrowserMessageWriter,
} from "vscode-languageserver-protocol/browser.js";
import EditorWorker from "monaco-editor/esm/vs/editor/editor.worker?worker";

export default {
  name: "MonacoEditor",
  data() {
    return {};
  },
  async mounted() {
    this.files = await this.readZipFile(
      new URL("./stdlib-source-with-typeshed-pyi.zip", window.location.href)
        .href
    );

    const pythonWorkerUrl = new URL(
      "../../node_modules/@typefox/pyright-browser/dist/pyright.worker.js",
      window.location.href
    ).href;

    const pyrightWorker = new Worker(pythonWorkerUrl);
    const transports = {
      reader: new BrowserMessageReader(pyrightWorker),
      writer: new BrowserMessageWriter(pyrightWorker),
    };
    // Launch pyright language server
    pyrightWorker.postMessage({
      type: "browser/boot",
      mode: "foreground",
    });

    // ------------[ Monaco Editor Wrapper config ]------------
    const wrapperConfig = {
      editorAppConfig: {
        $type: "classic",
        useDiffEditor: false,
        codeResources: {
          main: {
            text: 'def test():\n  print("Hello Python")',
            fileExt: "py",
            enforceLanguageId: "python",
          },
        },
        editorOptions: {
          language: "python",
          //language: "javascript",
          minimap: {
            enabled: true,
          },
          theme: "vs-dark",
        },
      },
      serviceConfig: {
        userServices: {
          ...getThemeServiceOverride(),
          ...getTextmateServiceOverride(),
          ...getConfigurationServiceOverride(),
        },
        debugLogging: true,
      },
    };
    const languageClientConfig = {
      name: "Pyright Language Client",
      languageId: "python",
      options: {
        $type: "WorkerDirect",
        worker: pyrightWorker,
      },
      clientOptions: {
        documentSelector: ["python"],
        errorHandler: {
          error: () => ({ action: ErrorAction.Continue }),
          closed: () => ({ action: CloseAction.DoNotRestart }),
        },
        initializationOptions: {
          files: this.files,
        },
      },
      connectionProvider: {
        get: () => Promise.resolve(transports),
      },
    };
    const loggerConfig = {
      enabled: true,
      debugEnabled: true,
    };

    useWorkerFactory({
      ignoreMapping: true,
      workerLoaders: {
        editorWorkerService: () => new EditorWorker(),
      },
    });

    const wrapper = new MonacoEditorLanguageClientWrapper();
    const userConfig = { wrapperConfig, languageClientConfig, loggerConfig };

    await wrapper.initAndStart(
      userConfig,
      document.getElementById("monaco-editor-root")
    );

    // debugger;

    // await initServices({
    //   serviceConfig: {
    //     userServices: {
    //       ...getThemeServiceOverride(),
    //       ...getTextmateServiceOverride(),
    //       ...getConfigurationServiceOverride(),
    //     },
    //     debugLogging: true,
    //   },
    // });

    // await whenReady();

    // monaco.editor.create(document.getElementById("monaco-editor-root"), {
    //   autoIndent: "full",
    //   fixedOverflowWidgets: true,
    //   value: `print("Hello World")`,
    //   automaticLayout: true,
    //   language: "python",
    //   theme: "vs-dark",
    //   minimap: {
    //     enabled: false,
    //   },
    //   wordBasedSuggestions: "off",
    //   wordWrap: "on",
    // });

    // updateUserConfiguration(`{
    //   "workbench.colorCustomizations": {
    //         "editor.background": "#fff",
    //     }
    // }`);

    // console.log(vscode.Uri.file("/workspace/hello.py"));

    // // const fileSystemProvider = new RegisteredFileSystemProvider(false);
    // // fileSystemProvider.registerFile(
    // //   new RegisteredMemoryFile(
    // //     vscode.Uri.file("/workspace/hello.py"),
    // //     'print("Hellosssss, World!")'
    // //   )
    // // );
    // // registerFileSystemOverlay(1, fileSystemProvider);

    // const pythonWorkerUrl = new URL(
    //   "../../node_modules/@typefox/pyright-browser/dist/pyright.worker.js",
    //   window.location.href
    // ).href;
    // console.info(`main.ts, pythonWorkerUrl: ${pythonWorkerUrl}`);
    // const worker = new Worker(pythonWorkerUrl);

    // //const worker = new PyrightWorker();

    // worker.postMessage({
    //   type: "browser/boot",
    //   mode: "foreground",
    // });
    // const reader = new BrowserMessageReader(worker);
    // const writer = new BrowserMessageWriter(worker);
    // const languageClient = this.createLanguageClient({ reader, writer });
    // languageClient.start();
    // reader.onClose(() => languageClient.stop());
  },

  methods: {
    async readZipFile(url) {
      try {
        const response = await fetch(url);
        const data = await response.arrayBuffer();
        const results = {};
        const zip = await JSZip.loadAsync(data);
        for (let [filename, file] of Object.entries(zip.files)) {
          if (filename.endsWith("/")) {
            continue;
          }
          filename = filename.replace(/^(stdlib|stubs)/, "/$1");
          results[filename] = await file.async("text");
        }
        return results;
      } catch (error) {
        console.error(error);
        return {};
      }
    },
    // createLanguageClient(transports) {
    //   return new MonacoLanguageClient({
    //     name: "Pyright Language Client",
    //     clientOptions: {
    //       documentSelector: ["python"],
    //       errorHandler: {
    //         error: () => ({ action: ErrorAction.Continue }),
    //         closed: () => ({ action: CloseAction.DoNotRestart }),
    //       },
    //       // workspaceFolder: {
    //       //   index: 0,
    //       //   name: "/workspace",
    //       //   uri: vscode.Uri.file("/workspace"),
    //       // },
    //       // synchronize: {
    //       //   fileEvents: [vscode.workspace.createFileSystemWatcher("**")],
    //       // },
    //       initializationOptions: {
    //         files: this.files,
    //       },
    //     },
    //     connectionProvider: {
    //       get: () => {
    //         return Promise.resolve(transports);
    //       },
    //     },
    //   });
    // },
  },
};
</script>

<style>
.editor-container {
  width: 100%;
  height: 100%;
}

.editor {
  width: 100%;
  height: 100vh;
}
</style>
