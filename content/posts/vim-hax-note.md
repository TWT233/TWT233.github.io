---
title: "Vim Hax Note"
date: 2021-12-14T00:12:17+08:00
draft: false
categories: ["life"]
tags: ["unix"]
---

~~老年人用~~想了想，老 hax 应该都肌肉记忆了，也用不到这个。那就老年新人用吧

variable: wrapped in \`\`, such as `n`

# navigation

|        command         |  key1  | key2 |
| :--------------------: | :----: | :--: |
|      back (word)       |   b    |      |
|     forward (word)     |   f    |      |
|      back (page)       | CTRL+b | PgUp |
|     forward (page)     | CTRL+f | PgDn |
|       line start       |   0    | home |
| line start (non-blank) |   ^    |      |
|        line end        |   $    | end  |

|       command       |  key1  |   key2    |
| :-----------------: | :----: | :-------: |
|     goto line n     | `n` gg |           |
|  mv up n line(col)  | `n` k  |  `n` Up   |
| mv down n line(col) | `n` j  | `n` Down  |
|    mv left n row    | `n` h  | `n` Left  |
|   mv right n row    | `n` l  | `n` Right |

# edit

|      command       |       key1       | key2 |
| :----------------: | :--------------: | :--: |
|        undo        |        u         |      |
|        redo        |    `n` CTRL+r    |      |
|    visual mode     |        v         |      |
| copy (visual mode) |        y         |      |
|       paste        |        p         |      |
|      replace       | :s/`old`/`new`/g |      |

# find

|      command      | key1 | key2 |
| :---------------: | :--: | :--: |
|    normal find    |  /   |      |
| find current word |  #   |      |
