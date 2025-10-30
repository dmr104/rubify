# Proof of concept

## Background information
This is the proof of concept for the project as the LSP (language server protocol) extension. 

VS Code Extension is an extension type that runs in the desktop or worker-based web environment.  We elect to use a desktop extension. This is also known as Electron.

VS code Electron is based upon Chromium plus Node, while VS Code Web is literally a browser

vscode.dev is Microsoft's hosted deployment of version of "VS Code For the Web".

vscode.dev is vscode running entirely in the browser, which is ideal for when we want no Electron and a sandboxed version of ruby-wasm for vscode.dev to run in the same browser runtime but with no DOM and no UI.

In Electron the renderer and the extension host are separate processes which communicate indirectly via VS Code's RPC over Electron or Node IPC.

Wasm stands for web-assembly.

A web-worker is a dedicated background JS (javascript) thread with no DOM (document object model).  

V8 is Google's open source JS/WebAssembly engine which runs JS with an interpreter, optimizing JIT, memory management with garbage collection.   

Each VS Code Window (Workbench UI) is its own Electron renderer process (own address space/V8 engine) with additional renderers for webviews and iframes. 

Each VS Code Window lives in a separate process space from the Electron main process, and from the per-window Extension Host Node process.

I don't need wasmtime runtime to run ruby-wasm within my bundled web-worker because I can just run ruby-wasm in a web-worker with the browser's WebAssembly + WASI (web assembly system interface) JS shim via the ruby.wasm runtime or @wasmer/ruby/wasix.  

An alternative approach would be to use a Node worker_threads worker by using node:wasi, which is a library which is within the Extension Host Node process: which utilises a separate V8 engine which isolates code within each thread so that the each Workbench UI renderer gets it's own isolated sandboxed VM (virtual machine). If I were to go with the node:wasi within the Extension Host Node process approach, then the node:wasi worker_threads would tay inside the Extension Host Node process and communicate with the Workbench UI renderer process via VScode's RPC layer (IMessagePassingProtocol), with the Workbench UI renderer process brokering channels. The instructions about this approach are too complicated, and also I would need to lock down the process to prevent require/fs/child_processes.  This might lead to a security hole if I forget something or a vulnerability is found.  Therefore I reject this approach as using node:wasi library inside of Extension Host Node process, in favour of running ruby-wasm inside of a bundled web-worker.

The LSP Client runs in the extension host with no DOM access at all.

The windows' renderer (Workbench UI renderer) processes own all the DOM and UI within each window and communicate with the extension host node process (which runs the LSP client as a library) via vscode's builtin renderer.  

In Electron desktop, which process-space do the DOM and UI of the open tabs reside? Answer: The DOM and UI for the workbench and editor tabs live in the window's renderer process (one per VS Code window), while embedded webviews and notebooks run in their own isolated renderer via a webview iframe sandbox.  

The Workbench UI renderer is the same as the Window's renderer process, also known as the BrowserWindow. The workbench UI renderer is always a sandboxed Node-disabled renderer, separate from Electron's main Node process.  The Workbench UI renderer talks to Electron's main Node process and the Extension Host Node process via IPC. 

The Extension Host Node process and the Workbench UI renderer are separate processes with isolated heaps. The latter (the Workbench UI renderer) is node-disabled, and the two talk only through the Electron's main Node process via IPC, and RPC over IPC such as Electron channels and MessagePorts. They have no shared process space.

Electron's main process and the Workbench UI renderer are isolated address spaces (no shared memory or heap or globals).  They communicate purely via IPC.  The same is said for Electron's main process and the Extension Host Node process: they are isolated processes with separate heaps and address spaces.  The EH (Extension Host) is spawned by main and they only talk via IPC (stdio/pipe/scokets/MessagePort), not via shared process space.

The Electron Node main process orchestrates booting the app, creating BrowserWindows which is owns, spawning the Extension Host and shared processes, wiring the IPC, handling OS native stuff (menus, dialogs, file handlers, protocol handlers), crash reporting, registering custom themes, enforcing permissions, and brokering privileged services to renderers. 

The LSP Client is just a JS/TS library (vscode-languageclient) that is loaded by the Extension Host Node Process and is executed there.  The LSP client does not require a separate process. 

In Electron desktop, is the Workbench UI Renderer within a different process space than the LSP client web-worker is? Answer: Yes, because the Workbench UI renderer is a Chromium renderer, while the LSP client lives within a different Extension Host Node.js process on Electron desktop.  They have no shared DOM at all, and must communicate via VS Code's own IPC channel, which will be the same as MessagePorts tunnelled via ELectron IPC.  

Launching code boots Electron's main process, which then creates the initial BrowserWindow renderer for the workbench (and more windows as are needed). So the Electron main process and the Workbench UI renderer are created automatically when VS code starts.

Recall that the LSP client runs as a library inside of the Extension Host Node process. 

So we will have just one LSP server. The ruby-wasm helper workers are not LSP servers.  They just post messages back to the LSP server, and it is only this server that speaks JSON-RPC to the in-host (within the Extension Host Node process) LSP client; which in turn communicates with the Workbench UI process via Electron-mediated IPC.  At startup, the Electron main process spawns the Extension Host Node process, and wires a duplex channel (named pipe/IPC) that carries VS Code's multiplexed JSON-RPC-style messages (structured-clone, MessagePort handoffs, cancellation, etc).

## My aims
Our aim is that the client of LSP (written in typescript) will spawn a server in another process, and this server will be a web-worker that embeds a shipped copy of ruby-wasm.  The LSP client will be running in a separate process than the Workbench UI renderer.

Within this extension we hope to run the LSP server logic: e.g. load ruby-wasm inside the web-worker and yet be able to talk JSON-RPC (javascript object notion - remote procedural call) to and from the client via postMessage/MessagePort.

We aim to forward requests to the ruby-wasm from the TS (typescript) server for static analysis.  These symbol tables, variable and constant names with scope resolution, types, control flow and data flows, call graphs, refs, lints, etc are called "static" because they are information about the program gleaned from ruby-wasm without the program being executed.  ruby-wasm parses code to the AST (abstract syntax tree) which is an IR (intermediate representation). An IR is a compiler friendly form of the program.  

A ruby interpreter parses to an AST and then compiles to YARV bytecode, and can do this JIT (just in time) via MJIT/YJIT before executing. IR is in three-address code, which is a simple form of an IR with steps like t1 = a + b that compilers use for optimization.

A SSA (static single assignment) whereby each variable gets one definition which enables good optimizations. A LLVM (low level virual machine) is a compiler-based toolchain whose IR is SSA-based.

ruby-wasm is an implementation of all of the above.

So the "static" analysis is called such because it parses ruby code to an AST, which is an IR that uses name/scope/type resolution, call-graphing and other abstract interpretation extracted via analyzing the ruby program, where the "ruby program" is as a higher level program written in ruby.

How do I ensure that the LSP web-server is running inside of a web-worker in order to run ruby-wasm?  Well, I have to bundle the LSP server code (including ruby-wasm initialisation) into a Worker entry script, and then in the client use "vscode-languageclient/browser" with various ServerOptions and MessagePort transport; while the LSP Server (running in that web-worker) uses "vscode-languageclient/browser" with BrowserMessageReader/Writer(self) and loads ruby-wasm there (see Microsoft language-server-protocol-in-web-extensions, and github microsoft/vscode-extension-samples/lsp-web-extension samples).

### Implementation plan
(As an aside: to simply create a LSP client and a server which communicate over LSP: In the VS Code extension host, create a client by using "vscode-languageclient".  Then spawn the server using IPC.)

But we really want to add a browser entry point in package.json with metadata.  keep "main": "dist/node/extension.js" for desktop, add "browser": "dist/node/extension.js" and "extensionKind": ["web", "workSpace"], and build a separate web bundle (which is a browser-targeted build of the extension) via esbuild that uses "vscode-languageclient/browser" and a worker-MessagePort for the JSON-RPC.


## Node IPC (inter process communication)

We first examine the set-up of the IPC between the Electron main process and the Extension Host Node process.

We are using Electron IPC with child Node process pipes. 

After the Electron's main process has spawned the extension host Node process, and we have wired an IPC Channel via child_process.fork (as a net socket or a named pipe), VS Code then multiplexes logical "channels" over this IPC.  The sandboxed workbench UI renderer talk back to the Electron main process posts async messages via ipcRenderer.send to ipcMain.  Many RPC channels are multiplexed over that single port. 

### RPC over Electron
This internal RPC is a typed proxy layer that multiplexes many logical channels over IPC via proxies.  The Electron main process bridges a single pipe or socket to the VS Code extension host node process where the LSP client runs, and then multiplexes all RPC over it.  Messages are length prefixed and framed with marshalled arguments (Uri, buffer, handles).  

## To examine how the LSP client and server communicate
MessagePort is one endpoint of a MessageChannel. The channel is the pair of linked endpoints, which gives us FIFO (first in first out) full-duplex messaging.

ruby-wasm isn't a web-worker if it be considered by itself, but it is a runtime which is loaded either into the worker as the LSP server or a separate helper worker.  

The LSP server runs within a dedicated web-worker, and both it and the ruby-wasm web-worker sit within the same browser process but within isolated threads with separate heaps, which will have no direct access to each other's memory space unless we have a SharedArrayBuffer or transfer ArrayBuffers via postMessage.

postMessage is not a protocol, but is a web-worker API for serializing data via the structured-clone algorithm, and delivers it as queued events relying upon FIFO delivery within a given channel. 

Structured clone is the web-workers' deep-serializer behind postMessage that copies most values (Object/Array/Map/Set/Date/RegExp/BigInt/TypedArrays/DataView/Blob/File/ImageData), and yet doesn't clone functions/DOM nodes/Weakmap/WeakSet/Symbol, but can move transferables in a zero-copy way via a transfer list (ArrayBuffer, MessagePort, ImageBitmap, OffScreenCanvas); while SharedArrayBuffer is shared by reference only if it is CrossOriginIsolated.  Delivery is FIFO per messagePort and is not JSON-based.  So you define your own payload schema and use it to move transferables.  Basically, the LSP client and server speak JSON-RPC over a MessagePort (which will utilize a structured clone under the hood) and a structured clone is the LSP server's way of sending all the data through a postMessage from the LSP client to ruby-wasm.

ruby-wasm is as a web-worker, and LSP server is as a web-worker, but the LSP client by default is NOT a web-worker because, by default, the LSP client runs inside of the extension host context which is as the extension host node process: which (the latter, extension host node process) is NOT a dedicated web-worker that runs the VS Code extension host Node process within itself.

(As an aside: in VS Code Web, the "extension host" would be a Dedicated Web Worker, and the LSP client would run inside of this worker, while the spawned LSP server would run within a separate dedicated web-worker.  They would communicate via the postMessage over the MessagePort.)  

## Debugging
In order to create a debugging environment we need ruby-wasm debugging capabilities and breakpoint support. We want to be able to stop and start the execution of ruby code at breakpoints that the user shall set.  Does the ruby debugger (rdbg) work in the wasm context?

Well, TracePoint is a Ruby VM API inside ruby-wasm.  We can use Tracepoint.new(:line) via conditionals for the user breakpoints, and pause the execution of wasm by creating a loop to make the ruby code park itself until it is told (by the TS worker thread) to continue again. The TS drives this via promises involving the postMessage web-worker API for async message passing that serializes data, optionally transferring ArrayBuffers over the web-workers MessagePort as the LSP server and ruby-wasm are both within the same process space. "vscode-languageclient/browser" facilitates us to do this.  We are to bridge LSP server with the ruby-wasm via a message channel.

We can inspect the call stack via tp.binding and caller, read locals via binding.local_variables

But ruby.wasm has no thread support. This is a WASI (Web Assembly System Interface) limitation: rdbg likely won't work, and you'd need a custom DAP (debug adapter protocol) adapter communicating over postMessage to bridge the web-worker to the Extension Host node process via the VS Code DAP chain. 

A custom DAP adapter is a process that implements initialize, launch/attach, setBreakPoints, step, etc, in order to bridge VS code's debug UI with the runtime debugger over stdio/sockets/MessagePort.

## Handling concurrency
within ruby.wasm we cannot fork a new process as the sandboxed environment prohibits this, and Thread.new will raise a NotImplementedError as WASI has no thread creation API yet.  But we can statically prescan with tree-sitter-ruby (which is an npm grammar called from Node in the extension host) or Ripper (ruby stdlib that runs inside ruby-wasm) to discover Thread.new/Process.*, and after having discovered and flagged Thread/Process/Reactor sites and boundaries we can construct a graph which will be a mapping from each site to an isolated unit.  

Let us assume we take the Ripper approach. I am inclined to go with this approach as it is within ruby and I presume that ruby can understand ruby code better the TS can.

I will bundle a helper web-worker with a ruby-wasm running within it. The plan is to create as many instances of these as there are new Threads or processes within the ruby code.  A maximum number of processes should be set to avoid run-away loops which create too many new processes.

The discovery of a Thread.new or Process.new will cause the Extension Host Node Process to start a helper web-worker process. Each will have a dedicated port (MessagePort).  

We will want to have created ruby.wasm instances in the helper web-worker processes, and have each wired to to the LSP server for message passing.  We do this by using a port handoff. The helper web-worker's renderer makes a MessageChannel per helper.  postMessage portA to the LSP server web-worker, and postMessage portB to the helper web-worker.  They then communicate over that port.  For many threads communicating with one LSP server we will need per-helper port channels, source IDs to track who sent what, and backpressure handling (queue/throttle) if helper flood the LSP Server faster than it can handle. 

Then we monkey-patch those APIs in the LSP server web-worker sandbox to postMessage back to the Electron extension host on actual calls. 

On the first invocation, spawn web workers each running ruby-wasm with a read-only FS (file system) snapshot and a unit ID.  We wire these units via MessageChannel to emulate pipes and queues.  

We run a deterministic scheduler (e.g. FIFO or priority by callsite), enforce CPU/mem/time caps.  

We terminate these web-workers on scope exit or cancel, with retries and backoff for failures. These helper web-worker never communicate with the main Electron process but do so indirectly, via the LSP server, with the Extension Host Node process which is running the LSP client as a library inside of it. 

It looks like:

Renderer(Workbench UI) <=> Extension Host <=> LSP Client <=> LSP Server 

The first is a Chromium renderer (browser JS) with nodeIntegration off.  As for the second, Node runs in the separate Extension Host Node process, not in the workbench UI.  For the third, which is as a web extension, the LSP server runs within a browser web-worker's own renderer (no Node API, no own OS process).  The LSP server talks message port to the client 

Electron main is not in the data path. 







