visible
-- TITLE --
Folding in LunarVim
-- TAGS --
vim
editor
-- TLDR --
Working with TailwindCSS in heex-template can get unwieldy. If only I could hide some of the lines.
-- CONTENT --
# Folding in LunarVim

Working with TailwindCSS in `heex` templates is super nice, only the enormous amount of css-classes makes for long lines and huge files.
I was having an issue with some `div` not aligning as I would like, but since they contain so much content I could not see what was happening.

I know there is some way to hide lines, what is it called again, ah yes, folding.

Since my code is automatically formatted thanks to the nice heex formatted plugin. I can be sure it is correctly indented.

```elixir
:set foldmethod=indent
```

Oops, this folds the whole file, leaving me with only the opening and closing tags. I can use `zo` to open up a single level of folding and `zR` to unfold all.

Not quite there, when I now open up a file, it folds everything all the time. This is not what I want, I want to be able to fold only when I feel like it.

```elixir
:set foldlevel=99
```

Now only folds 99 levels in will get folded, but I can still fold manually.

<span>&#128515;</span>

To make the change permanent, add the following to my lunarvim configuration:

```elixir
-- Folding
vim.api.nvim_set_option("foldmethod", "indent")
vim.api.nvim_set_option("foldlevel", 99)
```

Unsure on how to use the folds, just press `z` and LunarVim will show you all possibilities.


