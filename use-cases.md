# fastly use case

```
...
catch_unwind -
Rust         |
Native frames| 
Rust ! ------+
<top>
```

The native frames "speak DWARF" and have no landing pads registered.
Important thing is that "native frames" do not try to intercept the
panic, they just want it to "pass through".

Requirement:

* some way to translate a Rust panic into native system
* some way to convert back

# lua use case

```
...
setjmp           -
Rust             |
Native frames    | 
Rust longjmp ----+
<top>
```

The native frames "speak DWARF" and have no landing pads registered.
Important thing is that "native frames" do not try to intercept the
panic, they just want it to "pass through".

Requirement:

* some way to translate a Rust panic into native system
* some way to convert back
# more generally

