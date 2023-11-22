# WebAssembly for Proxies (AssemblyScript SDK)

This is a friendly fork of https://github.com/solo-io/proxy-runtime/,
temporarily mantained to address one incompatibility between the SDK and
ngx_wasm_module.

This fork should no longer be needed once that is handled in the upstream.


## How to use this SDK

Create a new project:

```shell
npm init
npm install --save-dev assemblyscript
npx asinit .
```

Include `"use": "abort=abort_proc_exit"` to the `asconfig.json` file as part of
the options passed to `asc` compiler:
```
{
  "options": {
    "use": "abort=abort_proc_exit"
  }
}
```

Add `"@kong/proxy-wasm-sdk": "0.0.6"` to your dependencies then run `npm install`.


## Hello, World

Copy this into assembly/index.ts:

```ts
export * from "@kong/proxy-wasm-sdk/proxy";
import {
  RootContext,
  Context,
  registerRootContext,
  FilterHeadersStatusValues,
  stream_context
} from "@kong/proxy-wasm-sdk";

class AddHeaderRoot extends RootContext {
  createContext(context_id: u32): Context {
    return new AddHeader(context_id, this);
  }
}

class AddHeader extends Context {
  constructor(context_id: u32, root_context: AddHeaderRoot) {
    super(context_id, root_context);
  }
  onResponseHeaders(a: u32, end_of_stream: bool): FilterHeadersStatusValues {
    const root_context = this.root_context;
    if (root_context.getConfiguration() == "") {
      stream_context.headers.response.add("hello", "world!");
    } else {
      stream_context.headers.response.add("hello", root_context.getConfiguration());
    }
    return FilterHeadersStatusValues.Continue;
  }
}

registerRootContext((context_id: u32) => { return new AddHeaderRoot(context_id); }, "add_header");
```
## build

To build, simply run:
```
npm run asbuild
```

build results will be in the build folder. `untouched.wasm` and `optimized.wasm` are the compiled 
file that we will use (you only need one of them, if unsure use `optimized.wasm`).

## Run
Configure envoy with your filter:
```yaml
          - name: envoy.filters.http.wasm
            config:
              config:
                name: "add_header"
                root_id: "add_header"
                configuration: "what ever you want"
                vm_config:
                  vm_id: "my_vm_id"
                  runtime: "envoy.wasm.runtime.v8"
                  code:
                    local:
                      filename: /PATH/TO/CODE/build/optimized.wasm
                  allow_precompiled: false
```
