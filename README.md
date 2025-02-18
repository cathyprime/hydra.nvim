# Hydra.nvim

<!-- <img align="right" width="300" src="./hydra.png"> -->
<img align="right" width="300" src="https://user-images.githubusercontent.com/13056013/172239710-a18e3a2f-1b96-40f2-833e-c424f2962577.png" />

<!--
<p align="center">
  <img width="200" src="./hydra.png">
</p>
-->

This is the Neovim implementation of the famous [Emacs
Hydra](https://github.com/abo-abo/hydra) package.

## Description for Poets

Once you summon the Hydra through the prefixed binding (the body + any one head), all
heads can be called in succession with only a short extension.

The Hydra is vanquished once Hercules, any binding that isn't the Hydra's head, arrives.
Note that Hercules, besides vanquishing the Hydra, will still serve his original purpose,
calling his proper command. This makes the Hydra very seamless.

## Description for Pragmatics

Imagine you want to change the size of your current window. Vim allows you to do it with
`<C-w>+`, `<C-w>-`, `<C-w><`, `<C-w>>` bindings. So, you have to press
`<C-w>+<C-w>+<C-w>+<C-w><<C-w><<C-w><...` as many times as you need (I know about count
prefixes, but I was never fond of them). Hydra allows you to press `<C-w>` just once and
then get access to any `<C-w>...` bindings without pressing the prefix again:
`<C-w>+++++--<<<<`. Or buffer side scrolling: instead of `zlzlzlzlzlzl...` press
`zlllllllllhhhl` to freely scroll buffer left and right. Any key other than bind to
a hydra will stop hydra state and do what they should.

Hydra also allows assigning a custom hint to such group of keybindings to allows you an
easy glance at what you can do.

If you want to quickly understand the concept, you can watch
[the original Emacs Hydra video demo](https://www.youtube.com/watch?v=_qZliI1BKzI)

<!-- panvimdoc-ignore-start -->

- [Installation](#installation)
- [Creating a New Hydra](#creating-a-new-hydra)
- [Config](#config)
  - [Default Configuration](#default-configuration)
- [Hint](#hint)
  - [Hint Configuration](#hint-configuration)
- [Heads](#heads)
- [Colors](#colors)
  - [Amaranth](#amaranth)
  - [Blue and Teal](#blue-and-teal)
  - [Pink](#pink)
- [Hooks](#hooks)
  - [Meta Accessors](#meta-accessors)
- [Public methods](#public-methods)
- [Highlights](#highlights)
- [Keymap Utility Functions](#keymap-utility-functions)
- [Statusline](#statusline)
- [Limitations](#limitations)
- [How it works under the hood](#how-it-works-under-the-hood)

<!-- panvimdoc-ignore-end -->

## Installation

To install with [lazy.nvim](https://github.com/folke/lazy.nvim) use:

```lua
{
    "nvimtools/hydra.nvim",
    config = function()
        -- create hydras in here
    end
}
```

## Creating a New Hydra

To create a hydra you need to call Hydra's constructor with a parameters table of the
form:

```lua
local Hydra = require("hydra")
Hydra({
    -- string? only used in auto-generated hint
    name = "Hydra's name",

    -- string | string[] modes where the hydra exists, same as `vim.keymap.set()` accepts
    mode = "n",

    -- string? key required to activate the hydra, when excluded, you can use
    -- Hydra:activate()
    body = "<leader>o",

    -- these are explained below
    hint = [[ ... ]],
    config = { ... },
    heads = { ... },
})
```

The more complex fields are described below:

- [hint](#hint)
- [config](#config)
- [heads](#heads)

## Config

With this table, you can set the behavior of the whole hydra.

```lua
config = {
    -- see :h hydra-heads
    exit = false, -- set the default exit value for each head in the hydra

    -- decides what to do when a key which doesn't belong to any head is pressed
    --   nil: hydra exits and foreign key behaves normally, as if the hydra wasn't active
    --   "warn": hydra stays active, issues a warning and doesn't run the foreign key
    --   "run": hydra stays active, runs the foreign key
    foreign_keys = nil,

    -- see `:h hydra-colors`
    color = "red", -- "red" | "amaranth" | "teal" | "pink"

    -- define a hydra for the given buffer, pass `true` for current buf
    buffer = nil,

    -- when true, summon the hydra after pressing only the `body` keys. Normally a head is
    -- required
    invoke_on_body = false,

    -- description used for the body keymap when `invoke_on_body` is true
    desc = nil, -- when nil, "[Hydra] .. name" is used

    -- see :h hydra-hooks
    on_enter = nil, -- called when the hydra is activated
    on_exit = nil, -- called before the hydra is deactivated
    on_key = nil, -- called after every hydra head

    -- timeout after which the hydra is automatically disabled. Calling any head
    -- will refresh the timeout
    --   true: timeout set to value of 'timeoutlen' (:h 'timeoutlen')
    --   5000: set to desired number of milliseconds
    timeout = false, -- by default hydras wait forever

    -- see :h hydra-hint-hint-configuration
    hint = false,
}
```

### Default Configuration

The above discusses per-hydra configuration. But Hydra.nvim also allows you to set default
values for the `config` table. These defaults are automatically applied to new hydras, but
can still be overridden on a per-hydra basis.

This is useful for setting config that you want to apply to all of your hydras, like
common floating window borders, or common hooks.

_Only applies to hydras that are created after you call the setup method_

```lua
require('hydra').setup({
    debug = false,
    exit = false,
    foreign_keys = nil,
    color = "red",
    timeout = false,
    invoke_on_body = false,
    hint = {
        show_name = true,
        position = { "bottom" },
        offset = 0,
        float_opts = { },
    },
    on_enter = nil,
    on_exit = nil,
    on_key = nil,
})
```

## Hint

The hint for a hydra can let you know that it's active, and remind you of the hydra's
heads.

The string for the hint is passed directly to the hydra:

```lua
Hydra({
    hint = [[ some multiline string ]]
})
```

By default, a one line hint is generated and displayed in the cmdline. Heads and their
descriptions are placed in the order they were passed into the `heads` table. Heads with
`{ opts = { desc = false }}` don't appear in auto-generated hints.

Values in the hint string are parsed with the following rules:

- anything between `_` is considered a head, and will be highlighted with the
  corresponding head [color](#colors).
- `^` is treated as an empty char and can be used to help align the hint (normally used
  on lines that don't have as many underscores as lines above or below)
- `%{val}` is a dynamic value, a function named `val` is called and its return value
  inserted into the hint
  - Updated each time a head is called
  - Pass these functions to `config.hint.funcs` (discussed below)
  - There are built-in functions located
  [here](https://github.com/nvimtools/hydra.nvim/blob/main/lua/hydra/hint/vim-options.lua)

**Heads not in the manually created hint, will be automatically added to the bottom of the
hint window, following the same rules as auto-generated hint. You can avoid this with
`{ desc = false }`**

### Hint Configuration

The hint is configured with the `hint` key on the [config](#config) table. You can disable
the hint by setting this value to `false`.

```lua
Hydra({
    config = {
        -- either a table like below, or `false` to disable the hint
        hint = {
            -- "window" | "cmdline" | "statusline" | "statuslinemanual"
            --   "window": show hint in a floating window
            --   "cmdline": show hint in the echo area
            --   "statusline": show auto-generated hint in the status line
            --   "statuslinemanual": Do not show a hint, but return a custom status
            --                       line hint from require("hydra.statusline").get_hint()
            type = "window", -- defaults to "window" if `hint` is passed to the hydra
                             -- otherwise defaults to "cmdline"

            -- set the position of the hint window. one of:
            --    top-left   |   top    |  top-right
            --  -------------+----------+--------------
            --   middle-left |  middle  | middle-right
            --  -------------+----------+--------------
            --   bottom-left |  bottom  | bottom-right
            position = "bottom",

            -- Offset of the floating window from the nearest editor border
            offset = 0,

            -- options passed to `nvim_open_win()`, see :h nvim_open_win()
            -- Lets you set border, header, footer, etc etc.
            float_opts = {
                -- row, col, height, width, relative, and anchor should not be
                -- overridden
                style = "minimal",
                focusable = false,
                noautocmd = true,
            },

            -- show the hydras name (or "HYDRA:" if not given a name), at the
            -- beginning of an auto-generated hint
            show_name = true,
            
            -- if set to true, this will prevent the hydra's hint window from displaying
            -- immediately.
            -- Note: you can still show the window manually by calling Hydra.hint:show()
            -- and manually close it with Hydra.hint:close()
            hide_on_load = false,

            -- Table from function names to function. Functions should return
            -- a string. These functions can be used in hints with %{func_name}
            -- more in :h hydra-hint
            funcs = {},
        }
    }
})
```

## Heads

Each Hydra's head has the form:

```lua
{ head, rhs, opts }
```

Similar to the `vim.keymap.set()` function.

The `head` is the "lhs" of the mapping (given as a string). These are the keys you press
to perform the action.

The `rhs` is the action that gets performed. It can be a string, function or `nil`. when
nil, the action is a no-op.

The `opts` table is empty by default.

```lua
opts = {
    -- "When the hydra hides, this head does not stick out"
    -- Private heads are unreachable outside of the hydra state.
    private = false,

    -- When true, stops the hydra after executing this head
    -- NOTE:
    --   - All exit heads are private
    --   - If no exit head is specified, esc is set by default
    exit = false,

    -- Like exit, but stops the hydra BEFORE executing the command
    exit_before = false,

    -- when set to false, config.on_key isn't run after this head
    ok_key = true,

    -- string | false - value shown in auto-generated hint. When false, this key
    -- doesn't show up in the auto-generated hint
    desc = nil,

    -- same as the builtin map options
    expr = false, -- :h :map-expression
    silent = false, -- :h :map-silent

    -- \/ For Pink Hydras only \/ --

    -- allows binding a key which will immediately perform its action and not wait
    -- `timeoutlen` for a possible continuation
    nowait = false,

    -- Override `mode` for this head
    mode = "n",
}
```

## Colors

The `color` option is a shortcut for determining the `exit` and `foreign_keys` options. It
sets them in the following way:

| color    | values                             |
| -------- | ---------------------------------- |
| red      |                                    |
| blue     | exit = true                        |
| amaranth | foreign_keys = 'warn'              |
| teal     | foreign_keys = 'warn', exit = true |
| pink     | foreign_keys = 'run'               |

> [!NOTE]
> `Color` has higher precedence than the `exit` and `foreign_keys` options. 
> If the values specified in the `exit` or `foreign_keys` options conflict with the 
> ones implied by `color`, the `exit` or `foreign_keys` options are ignored.


Colors are also used to highlight heads in the hint, so you know how they will behave.

Each hydra head has a _basic_ associated color, red or blue, that determines whether or
not the hydra will continue after the head is called:

- reddish head will execute the command and continue the state
- blueish head will execute the command and stop the state

The hydra body can be one of five variants of the basic colors: amaranth, teal, pink, red,
blue. They (according to basic color) determine the default behavior of all the heads; and
determine what happens when a foreign key is pressed. The following table summarizes the
effects of the different colors.

| Body Color | Basic color | Executing NON-HEAD    | Executing HEAD |
| ---------- | ----------- | --------------------- | -------------- |
| amaranth   | red         | Disallow and Continue | Continue       |
| teal       | blue        | Disallow and Continue | Quit           |
| pink       | red         | Allow and Continue    | Continue       |
| red        | red         | Allow and Quit        | Continue       |
| blue       | blue        | Allow and Quit        | Quit           |

### Amaranth

The amaranth color wasn't chosen at random just because it is a variation of the color
red. There is some lore &mdash; according to
[Wikipedia](http://en.wikipedia.org/wiki/Amaranth):

> The word amaranth comes from the Greek word amaranton, meaning "unwilting" (from the
> verb marainesthai, meaning "wilt"). The word was applied to amaranth because it did not
> soon fade and so symbolized immortality.

Hydras with amaranth body are impossible to quit with any binding except a blue head.

### Blue and Teal

A blue hydra has little sense in Vim since it works exactly like standard Vim multi-key
keybinding with addition you can add a custom hint to it.

A teal hydra works the same way, except it blocks all keys which are not hydra heads,
which can be useful.

### Pink

Pink hydra is of a different nature. It is
a [key-layer](https://github.com/nvimtools/hydra.nvim/tree/main/lua/hydra/layer) inside,
so all keys except overwritten are work as usual. Even `[count]` prefixes.

## Hooks

There are three hooks currently, `on_enter`, `on_exit`, and `on_key`. These fire when
you'd expect, and they're set on a per hydra basis. The `on_enter` function is called in
such a way that gives you access to [meta-accessors](#meta-accessors).

### Meta Accessors

Inside a function passed as `on_enter`, the `vim.o`, `vim.go`, `vim.bo` and `vim.wo`
[meta-accessors](https://github.com/nanotee/nvim-lua-guide#using-meta-accessors) are
redefined to work the way you think they should. If you want some option value to be
temporary changed while Hydra is active, you need just set it with one of the
meta-accessors in the `on_enter` function... and that's it. No need to set it back in
`on_exit` function.

```lua
config = {
    on_enter = function()
       print('Hydra enter')
       vim.bo.modifiable = false -- temporarily set `nomodifiable` while Hydra is active
    end,
    on_exit = function()
       print('Hydra exit')
       -- No need to set modifiable back here
    end
}
```

## Public methods

- `Hydra:activate()` — activate the hydra programmatically
- `Hydra:exit()` — exit the hydra if it is active

## Highlights

Hydra defines these highlight groups with their defaults colors:

- `HydraRed` &mdash; `fg = #FF5733`
- `HydraBlue` &mdash; `fg = #5EBCF6`
- `HydraAmaranth` &mdash; `fg = #ff1757`
- `HydraTeal` &mdash; `fg = #00a1a1`
- `HydraPink` &mdash; `fg = #ff55de`

- `HydraHint` — linked to `NormalFloat`, defines the fore- and background of the hint
  window;
- `HydraBorder` — linked to `FloatBorder`, defines the fore- and background of the border.
- `HydraTitle` — linked to `FloatTitle`, hl for the window title
- `HydraFooter` — linked to `FloatFooter`, hl for the window footer (only in nvim 0.10.0+)

## Keymap Utility Functions

Utility functions to use in keymaps, required with `require("hydra.keymap-util")`

- `cmd(command)`

  Get a string and wrap it in `<Cmd>`, `<CR>`

  ```lua
  cmd("vsplit") == "<Cmd>vsplit<CR>"
  ```

- `pcmd(try_cmd, catch?, catch_cmd?)`

  Protected `cmd`. Examples explain better:

  ```
  pcmd("wincmd k", "E11", "close")
  ->  "<Cmd>try | wincmd k | catch /^Vim\%((\a\+)\)\=:E11:/ | close | endtry<CR>"

  pcmd("wincmd k", nil, "close")
  ->  "<Cmd>try | wincmd k | catch | close | endtry<CR>"

  pcmd("close")
  ->  "<Cmd>try | close | catch | endtry<CR>"
  ```

  See: `:help exception-handling`

  **params:**

  - `try_cmd` : `string`
  - `catch` : `string` (optional) — String of the form `E` + some digits, like `E12` or
    `E444`.
  - `catch_cmd` : `string` (optional)

  **return:** `string`

## Statusline

In the statusline module `require('hydra.statusline')` there are functions that can help
you to integrate Hydra in your statusline:

- `is_active()` — returns `true` if there is an active hydra;
- `get_name()` — get the name of an active hydra if it has it;
- `get_color()` — get the color of an active hydra;
- `get_hint()` — get an active hydra's statusline hint. Return not `nil` only when
  `config.hint` is set to `false` or when `config.hint.type == "statuslinemanual"`

## Limitations

`[count]` is not supported in a red, amaranth and teal hydras (see `:help count`). But
supported in pink hydra since it is
a [layer](https://github.com/nvimtools/hydra.nvim/tree/main/lua/hydra/layer).

## How it works under the hood

You can read about the internal mechanics in
[CONTRIBUTING.md](https://github.com/nvimtools/hydra.nvim/blob/main/CONTRIBUTING.md)

## Thanks

- [anuvyklack](https://github.com/anuvyklack/) for creating the original hydra.nvim that
  this is forked from
- The original Emacs [hydra](https://github.com/abo-abo/hydra), for the concept this is
  based on

<!-- vim: set tw=90: -->
