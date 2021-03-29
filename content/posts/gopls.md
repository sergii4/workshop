---
title: "Gopls"
date: 2021-03-13T20:24:51Z
tags: ["go", "gopls", "lsp", "vim", "neovim", "vscode"]
# keywords: ["go", "gopls", "lsp", "vim", "neovim", "vscode"]
keywords: 
    - go
    - gopls
    - lsp
    - vim
    - neovim
    - vscode
description: "Gopls"
images:
    - "https://media.giphy.com/media/HVbyRe2CnqzRPxUfVy/giphy.gif"
draft: false
---

# Gopls
## Intro
Mid of 2018, I decided to switch from Java to Go. Language change leads to change editor and whole ecosystem and workflow such as doing my job primarily from the terminal.

At the very beginning of my Go journey, I tried Vim as an editor(with [vim-go](https://github.com/fatih/vim-go)), but I gave up because of a double burden such as learning a new language and a new editor at the same time. I switched back to a product well-known in the Java world company - [JetBrains](https://www.jetbrains.com) - which makes tools for software developers. Their IDE, [GoLand](https://www.jetbrains.com/go/) provides effortless immersion in the Go world.

After I got familiar with Go itself I decided to give Vim another try. I still have been using it as my primary editor.  From time to time I have to work with other languages(Javascript/NodeJS) and the question of how can I customize vim to work with any language arises.

I had a problem with Vim and Go as well. Vim-go itself is an umbrella for a set of Go tools. Part of them are provided by Go directly, others were written by enthusiast. With transition to the [Go Modules](https://blog.golang.org/using-go-modules) some of them [guru](https://pkg.go.dev/golang.org/x/tools/cmd/guru) [stopped](https://github.com/fatih/vim-go/issues/2851) working.

It was my gateway to [gopls](https://github.com/golang/tools/blob/master/gopls/README.md) and [Language Server Protocol](https://microsoft.github.io/language-server-protocol/).

In this post I try to answer the next questions:

- What is gopls?
- What problems does it solve?
- How does it work?
- What features does it have?
- Does it work well?

## What is Gopls
`gopls` (pronounced "Go please") is the official Go  [language server](https://langserver.org/)  developed by the Go team. It provides IDE features to any  [LSP](https://microsoft.github.io/language-server-protocol/) -compatible editor.
You should not need to interact with gopls directly - it will be automatically integrated into your editor. The specific features and settings vary slightly by editor, so we recommend that you proceed to the documentation for your editor below.

From [gopls design documentation](https://github.com/golang/tools/blob/master/gopls/doc/design/design.md):

* gopls should **become the default editor backend** for the major editors used by Go programmers, fully supported by the Go team.
* gopls will be a **full implementation of LSP**, as described in the  [LSP specification](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/) , to standardize as many of its features as possible.
* gopls will be **clean and extensible** so that it can encompass additional features in the future, allowing Go tooling to become best in class once more.
* gopls will **support alternate build systems and file layouts**, allowing Go development to be simpler and more powerful in any environment.

## What problems does it solve

![before img](https://about.sourcegraph.com/gophercon-2019/gopls-editors-tools.png)

While Go has a number of excellent and useful command-line tools that enhance the developer experience, it has become clear that integrating these tools with IDEs can pose challenges.
Support of these tools has relied on the goodwill of community members, and they have been put under a large burden of support at times as the language, toolchain and environments change. As a result many tools have ceased to work, have had support problems, or become confusing with forks and replacements, or provided an experience that is not as good as it could be.

This is fine for tools used occasionally, but for core IDE features, this is not acceptable. Autocompletion, jump to definition, formatting, and other such features should always work, as they are key for Go development.
The Go team will create an editor backend that works in any build system. It will also be able to improve upon the latency of Go tools, since each tool will no longer have to individually run the type-checker on each invocation, instead there will be a long-running process and data can be shared between the definitions, completions, diagnostics, and other features.

## How does it work and what features does it have
### Implementation
**View/Session/Cache**
Throughout the code there are references to these three concepts, and they build on each other.
At the base is the *Cache*. This is the level at which we hold information that is global in nature, for instance information about the file system and its contents.

Above that is the *Session*, which holds information for a connection to an editor. This layer hold things like the edited files (referred to as overlays).

The top layer is called the *View*. This holds the configuration, and the mapping to configured packages.

The purpose of this layering is to allow a single editor session to have multiple views active whilst still sharing as much information as possible for efficiency. In theory if only the View layer existed, the results would be identical, but slower and using more memory.

It is well illustrated in [tools/session.go](https://github.com/golang/tools/blob/master/internal/lsp/cache/session.go)

```
type Session struct {

	cache *Cache

	id    string

	optionsMu sync.Mutex
	options   *source.Options

	viewMu  sync.Mutex
	views   []*View

	viewMap map[span.URI]*View

	overlayMu sync.Mutex
	overlays  map[span.URI]*overlay

	// gocmdRunner guards go command calls from concurrency errors.

	gocmdRunner *gocommand.Runner
}
```

### Debug gopls
To get more information on how `gopls` works we need to make proper settings.

For Neovim([neovim/nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) plugin) in `init.vim`:
```
local lspconfig = require'lspconfig'
lspconfig.gopls.setup{
	cmd = {'gopls', '-logfile=/tmp/gopls.log', '-rpc.trace'},
}
```

For VS Code in [Go plugin](https://code.visualstudio.com/docs/languages/go) settings in `setting.json`
```
{
    "go.languageServerFlags": [
        "-logfile=/tmp/vs-gopls.log", "-rpc.trace"
    ]
}
```

Check log in `/tmp/gopls.log`  and `/tmp/vs-gopls.log`  respectively

### Open editor with Go project
When we open editor with open Go project next job should be done:
- starting server
- initialization
- open session and make session handshake
- works in progress: setting up workspace folder base on ([go.mod/go.sum](https://github.com/golang/tools/blob/master/gopls/doc/workspace.md)), loading packages (progress notifications send to an editor

From 
```
[Trace - 15:28:21.891 PM] Sending request 'initialize - (1)'.
```
We see:

worksapce folders and root path:
```
"workspaceFolders":[
      {
         "uri":"file:///Users/sgetman/code/me/workstation/test-go-prj",
         "name":"/Users/sgetman/code/me/workstation/test-go-prj"
      }
   ],
   "rootPath":"/Users/sgetman/code/me/workstation/test-go-prj",
```
clientInfo:
```
"clientInfo":{
      "version":"0.5.0",
      "name":"Neovim"
   },
```
and
```
"clientInfo":{
      "name":"Visual Studio Code",
      "version":"1.52.1"
   },
```
The we see group of capabilities:
```
"capabilities": {
    "workspace": {...},
    "textDocument": {...},
    "window": {...},
}
```

We take a look closet at `textDocument/definition`, `textDocument/hover` , `textDocument/codeActions` and `textDocument/completion`
![capabilities img](/img/capabilities.png)

### Open document, definition, completion, hover and code actions

To understand how `gopls` works we need to take a look at LSP.
From the [LSP Overview](https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/):
> a language server runs as a separate process and development tools communicate with the server using the language protocol over JSON-RPC. Below is an example for how a tool and a language server communicate during a routine editing session:  



![language server sequence img](/img/language-server-sequence.png)
To play around I use my simple [Go project](https://github.com/sergii4/workstation/tree/master/test-go-prj)
```
  1 package main
  2
  3 import (
  4         "fmt"
  5 )
  6
  7 func plus(a int, b int) int {
  8         return a + b
  9 }
 10
 11 func plusPlus(a, b, c int) int {
 12         return a + b + c
 13 }
 14
 15 func main() {
 16         res := plus(1, 2)
 17         fmt.Println("1+2 =", res)
 18
 19         res = plusPlus(1, 2, 3)
 20         fmt.Println("1+2+3 =", res)
 21 
```
Neovim
![go modules img](https://media.giphy.com/media/HVbyRe2CnqzRPxUfVy/giphy.gif)

VSCode
![go modules img](https://media.giphy.com/media/sMy9fZGDrr0fCfaJUK/giphy.gif)

I go to string *16* (in request it is line *15*, probably numeration starts with 0) and put cursor to the sumbol `p` of func `plus`  and invoke `go to definition`:

```
[Trace - 17:33:37.180 PM] Sending request 'textDocument/definition - (2)'.

{
   "textDocument":{
      "uri":"file:///Users/sgetman/code/me/workstation/test-go-prj/main.go"
   },
   "position":{
      "character":8,
      "line":15
   }
}

[Trace - 17:33:37.181 PM] Received response 'textDocument/definition - (2)' in 0ms.

[
   {
      "uri":"file:///Users/sgetman/code/me/workstation/test-go-prj/main.go",
      "range":{
         "start":{
            "line":6,
            "character":5
         },
         "end":{
            "line":6,
            "character":9
         }
      }
   }
]
```

Go to the line 21 and put cursor after `fmt.` and invoke autocompletion:

Neovim
![completion nvim img](https://media.giphy.com/media/RujN2WV9q2cylgGaxk/giphy.gif)

VSCode
![completion vscode img](https://media.giphy.com/media/DzGBD8XgbWRA4c492f/giphy.gif)

```
[Trace - 17:42:22.008 PM] Sending request 'textDocument/completion - (6)'.

{
   "textDocument":{
      "uri":"file:///Users/sgetman/code/me/workstation/test-go-prj/main.go"
   },
   "position":{
      "character":5,
      "line":19
   }
}

[Trace - 17:42:22.011 PM] Received response 'textDocument/completion - (6)' in 3ms.

{
   "isIncomplete":true,
   "items":[
      {
         "label":"Errorf",
         "kind":3,
         "detail":"func(format string, a ...interface{}) error",
         "documentation":"Errorf formats according to a format specifier and returns the string as a\nvalue that satisfies error.\n\nIf the format specifier includes a %w verb with an error operand,\nthe returned error will implement an Unwrap method returning the operand. It is\ninvalid to include more than one %w verb or to supply it with an operand\nthat does not implement the error interface. The %w verb is otherwise\na synonym for %v.\n",
         "preselect":true,
         "sortText":"00000",
         "filterText":"Errorf",
         "insertTextFormat":1,
         "textEdit":{
            "range":{
               "start":{
                  "line":19,
                  "character":5
               },
               "end":{
                  "line":19,
                  "character":5
               }
            },
            "newText":"Errorf"
         }
      },
...
      {
         "label":"Stringer",
         "kind":8,
         "detail":"interface{...}",
         "documentation":"Stringer is implemented by any value that has a String method,\nwhich defines the ``native'' format for that value.\nThe String method is used to print values passed as an operand\nto any format that accepts a string or to an unformatted printer\nsuch as Print.\n",
         "sortText":"00024",
         "filterText":"Stringer",
         "insertTextFormat":1,
         "textEdit":{
            "range":{
               "start":{
                  "line":19,
                  "character":5
               },
               "end":{
                  "line":19,
                  "character":5
               }
            },
            "newText":"Stringer"
         }
      }
   ]
}

```

Undo autocompletion, put cursor to the `P`, beginning of  `Prinltn` and invoke hover:
Neovim:
![hover nvim img](/img/hover-nvim.jpg)

VSCode:
![hover vscode img](/img/hover-vscode.jpg)

```
[Trace - 18:01:02.759 PM] Sending request 'textDocument/hover - (9)'.

{
   "textDocument":{
      "uri":"file:///Users/sgetman/code/me/workstation/test-go-prj/main.go"
   },
   "position":{
      "character":5,
      "line":19
   }
}

[Trace - 18:01:02.761 PM] Received response 'textDocument/hover - (9)' in 1ms.

{
   "contents":{
      "kind":"markdown",
      "value":"```go\nfunc fmt.Println(a ...interface{}) (n int, err error)\n```\n\n[`fmt.Println` on pkg.go.dev](https://pkg.go.dev/fmt?utm_source=gopls#Println)\n\nPrintln formats using the default formats for its operands and writes to standard output\\.\nSpaces are always added between operands and a newline is appended\\.\nIt returns the number of bytes written and any write error encountered\\.\n"
   },
   "range":{
      "start":{
         "line":19,
         "character":5
      },
      "end":{
         "line":19,
         "character":12
      }
   }
}
```

This example illustrates how the protocol communicates with the language server at the level of document references (URIs) and document positions. These data types are programming language neutral and apply to all programming languages. The data types are not at the level of a programming language domain model which would usually provide abstract syntax trees and compiler symbols (for example, resolved types, namespaces, …). The fact, that the data types are simple and programming language neutral simplifies the protocol significantly. It is much simpler to standardize a text document URI or a cursor position compared with standardizing an abstract syntax tree and compiler symbols across different programming languages.

Go to next line and add code:
```
	os.Create("tmp")
```

Diagnostic says us:
```
undeclared name: os
```
If we invoke `code actions` , in Neovim we see
![code actions nvim img](/img/code-actions-nvim.jpg)

in VSCode:
![code actions vscode](/img/code-actions-vscode.png)

and in gopls logs:
```
[Trace - 19:56:53.239 PM] Received notification 'textDocument/publishDiagnostics'.

{

   "uri":"file:///Users/sgetman/code/me/workstation/test-go-prj/main.go",
   "diagnostics":[
      {
         "range":{
            "start":{
               "line":20,
               "character":1
            },
            "end":{
               "line":20,
               "character":10
            }
         },
         "severity":1,
         "source":"compiler",
         "message":"undeclared name: os"
      }
   ]
}

[Trace - 19:57:00.001 PM] Received response 'textDocument/codeAction - (2)' in 21ms.
               "edits":[
                  {
                     "range":{
                        "start":{
                           "line":4,
                           "character":0
                        },
                        "end":{
                           "line":4,
                           "character":0
                        }
                     },
                     "newText":"\t\"os\"\n"
                  }
               ]
            }
         ]
      }
   },
   {

      "title":"Organize Imports",
      "kind":"source.organizeImports",
      "edit":{
         "documentChanges":[
            {
               "textDocument":{
                  "version":0,
                  "uri":"file:///Users/sgetman/code/me/workstation/test-go-prj/main.go"
               },
               "edits":[
                  {
                     "range":{
                        "start":{
                           "line":4,
                           "character":0
                        },
                        "end":{
                           "line":4,
                           "character":0
                        }
                     },
                     "newText":"\t\"os\"\n"
                  }
               ]
            }
         ]
      }
   }
]

[Trace - 19:57:04.274 PM] Sending notification 'textDocument/didChange'.

{
   "contentChanges":[
      {
         "text":"package main\n\nimport (\n\t\"fmt\"\n\t\"os\"\n)\n\nfunc plus(a int, b int) int {\n\treturn a + b\n}\n\nfunc plusPlus(a, b, c int) int {\n\treturn a + b + c\n}\n\nfunc main() {\n\tres := plus(1, 2)\n\tfmt.Println(\"1+2 =\", res)\n\n\tres = plusPlus(1, 2, 3)\n\tfmt.Println(\"0+2+3 =\", res)\n\tos.Create(\"tmp\")\n}\n"
      }
   ],
   "textDocument":{
      "uri":"file:///Users/sgetman/code/me/workstation/test-go-prj/main.go",
      "version":4
   }
}

```

## Does it work well
Gopls is a default editor Go backend for the most popular editors.
To get started with gopls, install an LSP plugin in your editor of choice.
*  [VSCode](https://github.com/golang/vscode-go/blob/master/README.md) 
*  [Vim / Neovim](https://github.com/golang/tools/blob/master/gopls/doc/vim.md) 
*  [Emacs](https://github.com/golang/tools/blob/master/gopls/doc/emacs.md) 
*  [Atom](https://github.com/MordFustang21/ide-gopls) 
*  [Sublime Text](https://github.com/golang/tools/blob/master/gopls/doc/subl.md) 
*  [Acme](https://github.com/fhs/acme-lsp) 

## Conclusion
Gopls allows you get best Go developing experience in an editor of you choice.  Full support by the Go team means it is free, stable and reliable tool.	

