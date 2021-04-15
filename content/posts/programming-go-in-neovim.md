---
title: "Programming Go in Neovim"
date: 2021-04-14T14:43:13+01:00
tags: ["go", "gopls", "lsp", "vim", "neovim"]
keywords: ["go", "gopls", "lsp", "vim", "neovim"]
images:
    - /img/cover-progrm-go.png	
draft: false
---

*There are tons of articles on how to programming Go in vim, how to turn vim into IDE. The purpose of this article is to look closer at nvim as an LSP client, especially for Go.*

![cover](/img/cover-progrm-go.png)

## Intro
Nvim introduced `nvim-lspconfig`, a collection of common configurations for Neovim’s built-in  [language server client](https://neovim.io/doc/user/lsp.html) .  From that point `nvim` can be lsp client for any server that supports LSP specification.

My primary setup before was `vim` with [vim-go](https://github.com/fatih/vim-go). After Go Modules were introduced some of the dependencies stopped working. `vim-go` depends on a bunch of tools. Support of these tools has relied on the goodwill of community members, and they have been put under a large burden of support at times as the language, toolchain, and environments change. As a result, many tools have ceased to work, have had support problems, or become confusing with forks and replacements, or provided an experience that is not as good as it could be.

Moreover, I want to have the same experience with other languages without wasting time exploring new plugins and tools. 

That’s how combination `nvim + gopls` appeared on my radar.


## Installation
To use the new (still experimental) native LSP client in Neovim, make sure you  [install](https://github.com/neovim/neovim/wiki/Installing-Neovim)  the prerelease v0.5.0 version of Neovim (aka “nightly”), the nvim-lspconfig configuration helper [plugin](https://github.com/neovim/nvim-lspconfig), and check the  [gopls configuration section](https://github.com/neovim/nvim-lspconfig/blob/master/CONFIG.md#gopls)  there.

## Migration from Vim-go
Vim-go was my only setup for development Go in vim and my only setup for development in vim at all. That's why I am so interested to set up a new Go development environment.


[Vim-plug](https://github.com/junegunn/vim-plug) is used as a pluging manager. So `:PlugInstall` has to be executed.


Minimal configuration in `$XDG_CONFIG_HOME/nvim/` which is usually means `~/.config/nvim/` :
```
nvim
├── init.vim
└── lua
    └── lsp_config.lua

```
{{< gist sergii4 afd763bb378aec45aba17c20b3cf2115 >}}

Let's take vim-go tutorial to make a comparision:

```
go get github.com/fatih/vim-go-tutorial
```

### Run, Build, Test, Cover it

For all this action I currently use command line and Go tools and haven't found analogs in nvim lsp.

To be honest, the only feature I really from the list is test capabilities.

### Fix it

Let's introduce two errors by adding two compile errors:
```
var b = foo()

func main() {
	fmt.Println("Hello GopherCon")
	a
}
```
![fix it img](https://media.giphy.com/media/1u2ggpiLhxOzfDzQ1N/giphy.gif)

Here nvim enters the battlefield. We immediately get error messages from [diagnostics](https://microsoft.github.io/language-server-protocol/specification#diagnostic) without build.

You see available diagnostics and can navigate it back and forth.

```
  buf_set_keymap('n', '[d', '<cmd>lua vim.lsp.diagnostic.goto_prev()<CR>', opts)
  buf_set_keymap('n', ']d', '<cmd>lua vim.lsp.diagnostic.goto_next()<CR>', opts)
```

I find diagnostics features easy to use:
- no additional action(build)
- no additional window - errors are shown in the code file


### Edit it

**Format**
Let us start with a unformatted file:
```
package main

     import "fmt"

func main() {
 fmt.Println("gopher"     )
}
```
![format img](https://media.giphy.com/media/2M2hadbXsItwq4ILm1/giphy.gif)

To format it by shortcut you need to have key binding in your lsp config:
```
-- Set some keybinds conditional on server capabilities
if client.resolved_capabilities.document_formatting then
    buf_set_keymap("n", "ff", "<cmd>lua vim.lsp.buf.formatting()<CR>", opts)
  elseif client.resolved_capabilities.document_range_formatting then
    buf_set_keymap("n", "ff", "<cmd>lua vim.lsp.buf.range_formatting()<CR>", opts)
  end
```

To format it on save file:
```
autocmd BufWritePre *.go lua vim.lsp.buf.formatting()
```

**Imports**

Let’s print the “gopher” string in all uppercase. For it we’re going to use the strings package. Change the definition to:
```
fmt.Println(strings.ToUpper(“gopher”))
```
You will see error `undeclared name: strings` shown by diagnostics

![code action image](https://media.giphy.com/media/JYKF33tWuXnvU5VTSm/giphy.gif)

You need `strings` in your imports and there is two ways to solve it: [code action](https://microsoft.github.io/language-server-protocol/specification#textDocument_codeAction) triggered manually 

Code action:
```
buf_set_keymap('n', 'ga', '<Cmd>lua vim.lsp.buf.code_action()<CR>', opts)
```

and [on save](https://github.com/golang/tools/blob/master/gopls/doc/vim.md#neovim-imports)

```
 function goimports(timeoutms)
    local context = { source = { organizeImports = true } }
    vim.validate { context = { context, "t", true } }

    local params = vim.lsp.util.make_range_params()
    params.context = context

    -- See the implementation of the textDocument/codeAction callback
    -- (lua/vim/lsp/handler.lua) for how to do this properly.
    local result = vim.lsp.buf_request_sync(0, "textDocument/codeAction", params, timeout_ms)
    if not result or next(result) == nil then return end
    local actions = result[1].result
    if not actions then return end
    local action = actions[1]

    -- textDocument/codeAction can return either Command[] or CodeAction[]. If it
    -- is a CodeAction, it can have either an edit, a command or both. Edits
    -- should be executed first.
    if action.edit or type(action.command) == "table" then
      if action.edit then
        vim.lsp.util.apply_workspace_edit(action.edit)
      end
      if type(action.command) == "table" then
        vim.lsp.buf.execute_command(action.command)
      end
    else
      vim.lsp.buf.execute_command(action)
    end
  end
```

and
```
autocmd BufWritePre *.go lua goimports(1000)
```

**Snippets**

This feature is a bit complicated in configuration. You can configure snippets in a usual way snippets engine([UltiSnips](https://github.com/SirVer/ultisnips)) and snippets itself([honza/vim-snippets](https://github.com/honza/vim-snippets)).

But LPS support there even more snippets options available. Gopls has a bunch of experimental features and [experimental postfix completions](https://github.com/golang/tools/blob/master/gopls/doc/settings.md#completion) is one of them. 

Setup requires additional effort. First, you have to set your LSP config in a way that informs LSP server that you have snippet support. So when the server run it knows about your capabilities:

```
local capabilities = vim.lsp.protocol.make_client_capabilities()
capabilities.textDocument.completion.completionItem.snippetSupport = true

nvim_lsp.gopls.setup{
	capabilities = capabilities,
}
```

It will be visible from logs with
```
:lua vim.cmd('e'..vim.lsp.get_log_path())
```
(prerequisites: 
```
lua << EOF
vim.lsp.set_log_level(“debug”)
EOF
```
)

logs:
```
capabilities: {
   ...
   "textDocument ="{
      ...
      "completion ="{
         "completionItem ="{
            ...
            "snippetSupport = true",
         },
     }
     ...
}
```

![postfix snippets](https://media.giphy.com/media/5xesSEqNeZV5McW0hE/giphy.gif)

### Check it

Gopls can publish not only errors, but warnings as well. Possible warning depends on [analyzers](https://github.com/golang/tools/blob/master/gopls/doc/analyzers.md) we specify. For example, `shadow` analyzer:
```
nvim_lsp.gopls.setup{
	...
	    settings = {
	      gopls = {
		     analyses = {
		    	shadow = true,
		      },
            ...
		    },
	    },
}
```

![shadow img](/img/shadow.png)

A lot of analyzer are still experiment but I enjoy to use it - it allows you do not postpone to linter execution step

### Navigate it

Probably most important feature set for development along with code completion.

![go to definition](https://media.giphy.com/media/ETZWdHqe7FUqAHEi8b/giphy.gif)

Nvim as LSP client supports next type of navigation request:
```
    callHierarchy/incomingCalls
    callHierarchy/outgoingCalls
    textDocument/declaration*
    textDocument/definition
    textDocument/implementation*
    textDocument/references
    textDocument/typeDefinition*
```

Key binding for some of them

```
  buf_set_keymap('n', 'gD', '<Cmd>lua vim.lsp.buf.declaration()<CR>', opts)
  buf_set_keymap('n', 'gd', '<Cmd>lua vim.lsp.buf.definition()<CR>', opts)
  buf_set_keymap('n', '<space>D', '<cmd>lua vim.lsp.buf.type_definition()<CR>', opts)
  buf_set_keymap('n', 'gi', '<cmd>lua vim.lsp.buf.implementation()<CR>', opts)
  buf_set_keymap('n', 'gr', '<cmd>lua vim.lsp.buf.references()<CR>', opts)
```

### Completion

Completion is available out of the box. We just have to map it to vim `omnifunc` to make our `Ctrl+x`,`Ctrl+o` work:

```
  buf_set_option('omnifunc', 'v:lua.vim.lsp.omnifunc')
```
![completion](https://media.giphy.com/media/T6JlMGWxQAha8lTvbv/giphy.gif)

### Understand it

Documentation lookup nvim supports as LSP client:
```
    textDocument/documentHighlight
    textDocument/documentSymbol
    textDocument/hover
    textDocument/signatureHelp
```

Key binding for some of them

```
  buf_set_keymap('n', '<C-k>', '<cmd>lua vim.lsp.buf.signature_help()<CR>', opts)
  buf_set_keymap('n', 'K', '<Cmd>lua vim.lsp.buf.hover()<CR>', opts)

  -- Set autocommands conditional on server_capabilities
  if client.resolved_capabilities.document_highlight then
    vim.api.nvim_exec([[
      hi LspReferenceRead cterm=bold ctermbg=red guibg=LightYellow
      hi LspReferenceText cterm=bold ctermbg=red guibg=LightYellow
      hi LspReferenceWrite cterm=bold ctermbg=red guibg=LightYellow
      augroup lsp_document_highlight
        autocmd! * <buffer>
        autocmd CursorHold <buffer> lua vim.lsp.buf.document_highlight()
        autocmd CursorMoved <buffer> lua vim.lsp.buf.clear_references()
      augroup END
    ]], false)
  end
```

![hover, signature help and document highlight](https://media.giphy.com/media/qarke1OUAAh16Qd49Q/giphy.gif)

### Refactor it
**Rename identifiers**
From nvim documentaton among LSP requests/notifications defined by default:
```
  textDocument/rename
```

Key binding for it

```
  buf_set_keymap('n', '<space>rn', '<cmd>lua vim.lsp.buf.rename()<CR>', opts)
```
![rename](https://media.giphy.com/media/72zU6mxK3Tn9zYShwT/giphy.gif)

## Conclusions
After exploring `nvim lsp` capabilities I understood how mature the tool `vim-go` is. It perfectly covers Go development workflow.

At the same time, nvim as an LSP client looks solid and gopls as server matches it pretty smooth. Despite nvim with lsp support is in a release version and a lot of gopls features are experimental it is already advanced Go development environment and we can expect more in the future.

When it comes to basic setup, nvim requires just one plugin - `nvim-lspconfig` and using gopls for it requires just one config line.
```
require'lspconfig'.gopls.setup{}
```
