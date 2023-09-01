# Weztem Mux + Nvim
This plugin is inspired by [Vim Tmux Navigator](https://github.com/christoomey/vim-tmux-navigator),
it allows to seemingly navigate between Wezterm panes and Nvim windows.
It requires some minor configuration in the Wezterm side and setup keybindings in NeoVim.

The neovim plugin provides 4 functions that check if the movement between windows
is done at an edge and a move out of neovim needs to be forward to wezterm.
If the movement is performed across neovim windows these functions work as the usual
`<C-w>h`, `<C-w>j` (..) combinations

![Wezterm+Nvim](https://github.com/jonboh/wezterm-mux.nvim/blob/media/wezterm_mux_nvim.gif?raw=true)

## Installation

### With Packer:
```lua
use {"jonboh/wezterm-mux.nvim"}
```

At some point in your NeoVim config you'll need to bind the keys you use to navigate between windows
to the functions of this plugin

```lua
local mux = require("wezterm-mux")
vim.keymap.set("n", "<C-h>", mux.wezterm_move_left)
vim.keymap.set("n", "<C-l>", mux.wezterm_move_right)
vim.keymap.set("n", "<C-j>", mux.wezterm_move_down)
vim.keymap.set("n", "<C-k>", mux.wezterm_move_up)
vim.keymap.set("n", "<A-x>", "<C-w>q") -- some actions dont need from a specific function
```

### With Lazy:
```lua
  {
    'jonboh/wezterm-mux.nvim',
    config = function()
      local mux = require 'wezterm-mux'
      vim.keymap.set('n', '<C-h>', mux.wezterm_move_left)
      vim.keymap.set('n', '<C-l>', mux.wezterm_move_right)
      vim.keymap.set('n', '<C-j>', mux.wezterm_move_down)
      vim.keymap.set('n', '<C-k>', mux.wezterm_move_up)
      vim.keymap.set('n', '<A-x>', '<C-w>q') -- some actions dont need from a specific function
    end,
  },
```


Given that this plugin is quite small and doesn't have configuration 
it should work on any other plugin manager. Let me know if it does not.


## Wezterm Config
You should add this configuration snippet some place in your `wezterm/wezterm.lua`.

```lua
local wezterm = require("wezterm")
local act = require("wezterm").action
local mux = require("wezterm").mux

local mod_pane_move = 'CTRL'

local platform = {
  is_win = string.find(wezterm.target_triple, 'windows') ~= nil,
  is_linux = string.find(wezterm.target_triple, 'linux') ~= nil,
  is_mac = string.find(wezterm.target_triple, 'apple') ~= nil,
}

local function is_nvim(window)
  local current_process = mux.get_window(window:window_id()):active_pane():get_foreground_process_name()
  wezterm.log_info(current_process)
  if platform.is_win then
    return string.find(current_process, 'nvim')
  else
    local nvim = '/usr/bin/nvim' -- change this to the location of you nvim
    return current_process == nvim
  end
end

local wez_nvim_action = function(window, pane, action_wez, forward_key_nvim)
  if is_nvim(window) then
    window:perform_action(forward_key_nvim, pane)
  else
    window:perform_action(action_wez, pane)
  end
end

wezterm.on('move-left', function(window, pane)
  wez_nvim_action(
    window,
    pane,
    act.ActivatePaneDirection 'Left', -- this will execute when the active pane is not a nvim instance
    act.SendKey { key = 'h', mods = mod_pane_move } -- this key combination will be forwarded to nvim if the active pane is a nvim instance
  )
end)

wezterm.on('move-right', function(window, pane)
  wez_nvim_action(window, pane, act.ActivatePaneDirection 'Right', act.SendKey { key = 'l', mods = mod_pane_move })
end)

wezterm.on('move-down', function(window, pane)
  wez_nvim_action(window, pane, act.ActivatePaneDirection 'Down', act.SendKey { key = 'j', mods = mod_pane_move })
end)

wezterm.on('move-up', function(window, pane)
  wez_nvim_action(window, pane, act.ActivatePaneDirection 'Up', act.SendKey { key = 'k', mods = mod_pane_move })
end)

-- you can add other actions, this unifies the way in which panes and windows are closed
-- (you'll need to bind <A-x> -> <C-w>q)
wezterm.on('close-pane', function(window, pane)
  wez_nvim_action(window, pane, act.CloseCurrentPane { confirm = false }, act.SendKey { key = 'x', mods = 'ALT' })
end)

-- you can add other actions, this unifies the way in which panes and windows are closed 
-- (you'll need to bind <A-x> -> <C-w>q)
wezterm.on("close-pane", function (window, pane)
   wez_nvim_action(window, pane, 
    act.CloseCurrentPane {confirm=false},
    act.SendKey({key="x", mods="ALT"})
   )
end)


config = {
    -- your other configuration

    keys = {
        {
            -- your other key bindings

            -- pane move(nvim aware)
            { key = 'h', mods = mod_pane_move, action = act { EmitEvent = 'move-left' } },
            { key = 'l', mods = mod_pane_move, action = act { EmitEvent = 'move-right' } },
            { key = 'j', mods = mod_pane_move, action = act { EmitEvent = 'move-down' } },
            { key = 'k', mods = mod_pane_move, action = act { EmitEvent = 'move-up' } },
            { key = 'x', mods = 'ALT', action = act { EmitEvent = 'close-pane' } },
        }
    }
}

-- more config

return config
```

Make sure to modify the variable nvim to your nvim binary location. You can run this if you are unsure of its location:
```shell
which nvim
```

## Additional help
You can check the relevant files in my [wezterm config](https://github.com/jonboh/dotfiles/tree/main/.config/wezterm) and [neovim config](https://github.com/jonboh/dotfiles/blob/main/.config/nvim/after/plugin/keymap/init.lua#L36) from my dotfiles.

Or open an [issue](https://github.com/jonboh/wezterm-mux.nvim/issues).

## Related
- [aca/wezterm.nvim](https://github.com/aca/wezterm.nvim): Unfortunately this needs from an external binary to cross the nvim/wezterm boundary and it uses some deprecated neovim functionality.
- [willothy/wezterm.nvim](https://github.com/willothy/wezterm.nvim): You could get something similar with this plugin, but it lacks the wezterm specific instructions.
