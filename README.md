[![CircleCI](https://circleci.com/gh/snoe/clojure-lsp/tree/master.svg?style=svg)](https://circleci.com/gh/snoe/clojure-lsp/tree/master)

# clojure-lsp

A [Language Server](https://microsoft.github.io/language-server-protocol/) for Clojure. Taking a Cursive-like approach of statically analyzing code.

## What is this?

The goal of this project is to bring great editing tools for Clojure to all editors.
It aims to work alongside you to help you navigate, identify and fix errors, and perform refactorings.

You will get:

- **Autocomplete**
- **Jump to definition**
- **Find usages**
- **Renaming**
- **Errors**
- **Automatic ns management**
- **Refactorings**

This is an early work in progress, contributions are very welcome.

## Installation

- You need `java` on your $PATH.
- Grab the latest `clojure-lsp` from github [LATEST](https://github.com/snoe/clojure-lsp/releases/latest)
- Place it in your $PATH with a chmod 755
- Follow the documentation for your editor's language client. See [Clients](#clients) below.

## Troubleshooting

See [troubleshooting.md](docs/troubleshooting.md).

## Capabilities

| capability | done | notes |
| ---------- | ---- | ----- |
| completionProvider | √ | |
| referencesProvider | √ | |
| renameProvider     | √ | |
| definitionProvider |   | TODO: java classes |
| diagnostics        |   | TODO: wrong arity, others? |
| hover              | √ | |
| formatting         | √ | |

## Refactorings

It should be possible to introduce most of the refactorings here: https://github.com/clojure-emacs/clj-refactor.el/tree/master/examples
Calling executeCommand with the following commands and additional args will notify the client with `applyEdit`.
All commands expect the first three args to be `[document-uri, line, column]` (eg `["file:///home/snoe/file.clj", 13, 11]`)

| done | command             | args | notes |
| ---- | ------------------- | ---- | ----- |
|   √  | cycle-coll          | | |
|   √  | thread-first        | | |
|   √  | thread-first-all    | | |
|   √  | thread-last         | | |
|   √  | thread-last-all     | | |
|   √  | introduce-let       | `[document-uri, line, column, binding-name]` | |
|   √  | move-to-let         | `[document-uri, line, column, binding-name]` | |
|   √  | expand-let          | | |
|   √  | add-missing-libspec | | |
|   -  | clean-ns            | | | sort only

See Vim client section for an example.

Other clients might provide a higher level interface to `workspace/executeCommand` you need to pass the path, line and column numbers.

## InitializationOptions

It is possible to pass some options to clojure-lsp through clients' `InitializationOptions`. Options are a map with keys:

`source-paths` value is a vector of project-local directories to look for clj/cljc/cljs files. Default is `["src","test"]`.

`dependency-scheme` by default, dependencies are linked with vim's `zipfile://<zipfile>::<innerfile>` scheme, however you can use a scheme of `jar` to get urls compatible with java's JarURLConnection. You can have the client make an lsp extension request of `clojure/dependencyContents` with the jar uri and the server will return the jar entry's contents. [Similar to java clients](https://github.com/redhat-developer/vscode-java/blob/a24945453092e1c39267eac9367c759a6c7b0497/src/extension.ts#L290-L298)

`cljfmt` json encoded configuration for https://github.com/weavejester/cljfmt

```json
"cljfmt": {
  "indents": {
    "#.*": [["block", 0]],
    "ns": [["inner", 0], ["inner", 1]],
    "and": [["inner", 0]],
    "or": [["inner", 0]],
    "are": [["inner", 0]]
}},
```

`macro-defs` value is a map of fully-qualified macros to a vector of definitions of those macros' forms.


Valid element definitions are:
  - `{"element": "declaration", "tags", ["unused", "local"], "signature": ["next"]}` This marks a symbol or keyword as a definition. `tags` are optional. The `unused` tag supresses the "unused declaration" diagnostic, useful for `deftest` vars. The `local` tag marks the var as private. `signature` is optional - if the macro has `defn`-like bindings this vector of movements should point to the parameter vector or the first var arg list form (only `next` is supported right now).
    - e.g. `(my-defn- my-name "docstring" [& params] (count params))` => `{"my.ns/my-defn-" [{"element": "declaration", "tags", ["local"], "signature": ["next" "next"]}]}`
  - `bindings` This marks `let` and `for`-like bindings. `bound-elements` will have these bindings in their scope.
    - e.g. `(my-with-open [resource ()] ....)` => `{"my.ns/my-with-open" ["bindings", "bound-elements"]}`
  - `function-params-and-bodies` This will parse function like forms that support optional var-args like `fn`.
    - e.g. `(myfn ([a] ...) ([b] ...)) (myfn [c] ...)` => `{"my.ns/myfn" ["function-params-and-bodies"]}`
  - `params` This marks a `defn` like parameter vector. `bound-elements` will have these parameters in their scope.
    - e.g. `(myfn [c] ...)` => `{"my.ns/myfn" ["params", "bound-elements"]}`
  - `param` This marks a single `defn` like parameter. `bound-elements` will have these parameters in their scope.
  - `elements` This will parse the rest of the elements in the macro form with the usual rules.
    - e.g. `(myif-let [answer (expr)] answer (log "no answer") "no answer")` => `{"my.ns/myif-let" ["bindings", "bound-element", "elements"]}`
  - `element` This will parse a single element in the macro form with the usual rules.
  - `bound-elements` This will parse the rest of the elements in the macro form with the usual rules but with any `bindings` or `params` in scope.
  - `bound-element` This will parse a single element in the macro form with the usual rules but with any `bindings` or `params` in scope.

See https://github.com/snoe/clojure-lsp/blob/master/test/clojure_lsp/parser_test.clj for examples.

## Clients

Clients are either editors with built in LSP support like Oni, or an appropriate plugin.
*Clients are responsible for launching the server, the server is a subprocess of your editor not a daemon.*

In general, make sure to configure the client to use stdio and a server launch command like `['/usr/local/bin/clojure-lsp']`.
If that fails, you may need to have your client launch inside a shell, so use someting like `['bash', '-c', '/usr/local/bin/clojure-lsp']`.
In windows you probably need to rename to `clojure-lsp.bat`.

### Vim
I prefer https://github.com/neoclide/coc.nvim but both http://github.com/autozimu/LanguageClient-neovim and https://github.com/prabirshrestha/vim-lsp work well.

See my [nvim/init.vim](https://github.com/snoe/dotfiles/blob/master/home/.vimrc) and [coc-settings.json](https://github.com/snoe/dotfiles/blob/master/home/.vim/coc-settings.json)

LanguageClient-neovim can be configure with:

Refactorings:
```vim

function! Expand(exp) abort
    let l:result = expand(a:exp)
    return l:result ==# '' ? '' : "file://" . l:result
endfunction

nnoremap <silent> crcc :call LanguageClient#workspace_executeCommand('cycle-coll', [Expand('%:p'), line('.') - 1, col('.') - 1])<CR>
nnoremap <silent> crth :call LanguageClient#workspace_executeCommand('thread-first', [Expand('%:p'), line('.') - 1, col('.') - 1])<CR>
nnoremap <silent> crtt :call LanguageClient#workspace_executeCommand('thread-last', [Expand('%:p'), line('.') - 1, col('.') - 1])<CR>
nnoremap <silent> crtf :call LanguageClient#workspace_executeCommand('thread-first-all', [Expand('%:p'), line('.') - 1, col('.') - 1])<CR>
nnoremap <silent> crtl :call LanguageClient#workspace_executeCommand('thread-last-all', [Expand('%:p'), line('.') - 1, col('.') - 1])<CR>
nnoremap <silent> crml :call LanguageClient#workspace_executeCommand('move-to-let', [Expand('%:p'), line('.') - 1, col('.') - 1, input('Binding name: ')])<CR>
nnoremap <silent> cril :call LanguageClient#workspace_executeCommand('introduce-let', [Expand('%:p'), line('.') - 1, col('.') - 1, input('Binding name: ')])<CR>
nnoremap <silent> crel :call LanguageClient#workspace_executeCommand('expand-let', [Expand('%:p'), line('.') - 1, col('.') - 1])<CR>
nnoremap <silent> cram :call LanguageClient#workspace_executeCommand('add-missing-libspec', [Expand('%:p'), line('.') - 1, col('.') - 1])<CR>
```

`InitializationOptions` can be sent by setting:
`let g:LanguageClient_settingsPath=".lsp/settings.json"`

Project-local `.lsp/settings.json` would have content like:
```
{"initializationOptions": {
   "source-paths": ["shared-src", "src", "test", "dashboard/src"],
   "macro-defs": {"project.macros/dofor": ["bindings", "bound-elements"]}}}
```

### Oni
Seems to work reasonably well but couldn't get rename to work reliably https://github.com/onivim/oni

### Intellij / Cursive
https://github.com/gtache/intellij-lsp tested only briefly.

### vscode
Proof of concept in the client-vscode directory in this repo.

### atom
I tried making a client but my hello world attempt didn't seem to work. If someone wants to take this on, I'd be willing to package it here too.

### emacs
Using https://github.com/emacs-lsp the following works for registering clojure-lsp:

```
(require 'lsp-mode)
(lsp-register-client
 (make-lsp-client :new-connection (lsp-stdio-connection '("bash" "-c" "clojure-lsp"))
                  :major-modes '(clojure-mode clojurec-mode clojurescript-mode)
                  :server-id 'clojure-lsp))
(add-to-list 'lsp-language-id-configuration '(clojure-mode . "clojure-mode"))
(setq lsp-enable-indentation nil)
(add-hook 'clojure-mode-hook #'lsp)
(add-hook 'clojurec-mode-hook #'lsp)
(add-hook 'clojurescript-mode-hook #'lsp)
```

## TODO

### Diagnostics
- configuration (see joker lint options)

### Others
- Better completion item kinds and auto require
- other lsp capabilities?
