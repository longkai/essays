Advanced Editing
===

## Customizing vi

### The :set command
``:set option`` to turn a toggle option on
``:set nooption`` to turn a toggle option off
``:set all`` display complete list of options
``:set option?`` find out the option by name
``:set`` shows options that you have specifically changed

### Executing Unix Commands
``:!command``
``:(n)r !command`` 

### Saving Commands
``:ab abbr phrase``
``:unab abbr``
``:ab``

``:map x sqquence``
``:unmap x``
``:map``

the keys not used in command mode that available for user-defined commands:
1. Letters: g, K, q, V, v
2. Control kyes: ^A, ^K, ^O, ^W, ^X
3. Symbols: _, *, \, =

