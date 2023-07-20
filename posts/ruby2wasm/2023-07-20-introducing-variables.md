---
title: Ruby to WASM Compiler - Introducing Variables
layout: default.liquid
is_draft: false
---
Currently our compiler only understands strings sent directly the print function, and it will generate a print "syscall" for each argument. The next step is to be able to store information into variables and then print those variables.

To the programmer this isn't much different but it is hugely different to our current compiler, which just stores the string to print with the call to `print` in our fledgling intermediate representation.

## A "Proper" IR

Intermediate Representation, or IR, is the middle step between parsed input and output. This is how proper compilers work, with the IR separating the frontend (parsing and lexing) from the backend (optimisation and code emitting). This is also how LLVM deals with so many languages. Language designers only need to translate their language into the LLVM IR and they reap the benefits of years of optimisation.

Not us though. That would be cheating[^1].

If we look at the code so far, and think hard, there are three things that our IR will need to do. The two are obvious; call `print` and store byte data. The third I struggled with though. Given a list of arguments we need to be able to create a list of `ciovec`. I kept thinking it would be easiest to just convert into the `ciovec` structure immediately like we do with the current version, but I lost a lot of time trying.

After I came to the realisation I needed the convert call, a lot of stuff fell into place. After all part of the point of having an IR is to separate out logic from parsing and code emitting so having the parsing stage create extra objects is obviously going to cause headaches.

### SideQuest 1: Inspecting the IR so far

If we are going to be working on an IR it would be useful if we could inspect it directly. This is pretty easy to on the current compiler. If we just print out the body of the `_start` function, by changing the write to file logic the end of our compiler with something like this
```rust
    let output = match args.mode {
        OutputModes::Wat => module.as_wat(constants),
        OutputModes::Ir => format!("{:#?}", module.functions[0].body)
    };

    if let Some(output_file) = args.output {
        std::fs::write(output_file, output).expect("Unable to write to output file");
    } else {
        println!("{}", output);
    }
```

With an added sprinkle of `#[derive(Debug)]` around the rest of the compiler and adding the `--mode` argument to clap we get:
```
$ cargo run --quiet -- -i src/test_model.rb --mode ir
[
    InvokePrint {
        variable: 8,
    },
]
```

Not super interesting, but neither is the original ruby source file. The compiler at this point is [here](https://github.com/xulaus/ruby2wasm/blob/23542e9/src/main.rs).

## Making an IR that works for us

Lets replace the current parsing logic with what we the IR we thought about and designed above. Instead of creating a bunch of CIOVecs and pushing them to the constant registry, we can simply push the following
```rust
current_function.body.extend_from_slice(&[
    IR::StoreData {
        variable: cur_varid,
        bytes: Box::from(raw_s.value.as_raw().as_slice()),
    },
    IR::ToCioVecArray {
        variable_ptr: cur_varid + 1,
        variable_length: cur_varid + 2,
        arguments: Box::new([cur_varid]),
    },
    IR::CallPrint {
        arguments: [cur_varid + 1, cur_varid + 2],
    },
])
```
In something like ruby, this means the print statement becomes
```ruby
const = "Hello"
ptr = [const]
length = 1

print(ptr, length)
```

That much is great, but now `WasmFunction` contains a bunch of `IR` nodes instead of a bunch of `WasmInstructions`. So how do we convert from `IR` to WAT? First thing we need our `WasmInstructions` to become a little more descriptive. Instead of just `InvokePrint` lets add some more instructions.

```rust
enum WasmInstruction {
    PushConst(i32),
    PushOffset(VariableId),
    Call(&'static str),
    Drop,
}
```

`Call` and `Drop` both line up with their WAT meanings, `PushConst` here is the same as `i32.const x`. `PushOffset` is a little bit subtler. I didn't want to loose the flexibility of being able to move data about just yet, so `PushOffset` tells the code emitter to ask for the location of a heap variable in memory before emitting the `i32.const`. It's almost like a grabbing a pointer in C.

Converting from these instructions into WAT is trivial, and hopefully it will be easy to write a translator to the binary format later. The conversion for each statement looks something like this (edited for clarity)
```rust
  match *i {
      PushConst(x) => wat.push_str(&format!("i32.const {x}\n")),
      PushOffset(x) => {
          let WasmVariable { offset, .. } = constants.fetch(x);
          wat.push_str(&format!("i32.const {offset}\n"))
      }
      Call(fun) => wat.push_str(&format!("call ${fun}\n")),
      Drop => wat.push_str(&format!("drop\n")),
  }
```

However the conversion from `IR` to `WasmInstruction` is a bit more involved. As we are now assigning to "variables" in the IR, we need a table to be able to look up the values of those variables. Luckily there are only 2 types of variable so far, a pointer to the heap, and an `i32`
```rust
#[derive(Debug, Copy, Clone)]
enum VariableTableEntry {
    Ptr(u32),
    Const(i32),
}
```
We'll probably have more trouble here in future when we want more types, especially more sizes of types, but for now this works.

We should now be able to write simple conversion routines. `StoreData` is trivial, and doesn't actually need to emit instructions
```rust
StoreData { variable, bytes, .. } => {
    variable_table.insert(*variable, Ptr(constants.push_const(bytes, 0)));
}
```

Surprisingly, neither does `ToCioVecArray`
```rust
ToCioVecArray {
    variable_ptr,
    variable_length,
    arguments,
} => {
    let mut ciovec_bytes: Vec<u8> = vec![0; arguments.len() * 8];
    for (i, arg) in arguments.iter().enumerate() {
        let Ptr(ptr) = variable_table[arg] else return Err(/* */)
        let var = constants.fetch(ptr);

        let (offset, len) = ciovec_bytes[8 * i..8 * i + 8].split_at_mut(4);
        offset.copy_from_slice(&var.offset.to_le_bytes());
        len.copy_from_slice(&var.len.to_le_bytes());
    }

    variable_table.insert(
        *variable_ptr,
        Ptr(constants.push_const(&ciovec_bytes, 4)),
    );
    variable_table.insert(*variable_length, Const(arguments.len().try_into().unwrap()));
}
```
I've removed some verbose error handling code and replaced it with an empty comment (`/* */`).

This leaves only `CallPrint` to emit any code at all
```rust
CallPrint {
    arguments: [variable_ptr, variable_len],
} => {
    // Get values with some type checking
    let Ptr(const_id) = variable_table[variable_ptr] else return Err(/* */)
    let Const(len) = variable_table[variable_len] else return Err(/* */)

    wasm.push(WasmInstruction::PushConst(1)); // push STDOUT
    wasm.push(WasmInstruction::PushOffset(const_id));
    wasm.push(WasmInstruction::PushConst(len));
    wasm.push(WasmInstruction::PushOffset(const_id)); // Write return value over input
    wasm.push(WasmInstruction::Call("print"));
    wasm.push(WasmInstruction::Drop);
}
```

All we then need to do is some minor changes for error handling, and to update how `WasmFunction` renders WAT
```rust
let body_wat = wasm_to_wat(&ir_to_wasm(&self.body, stack)?, indent_level + 2, stack);
```

And we are away! We can now check to see if our test ruby code has been interpreted correctly
```
$ cargo run --quiet -- -i src/test_model.rb --mode ir
[
    StoreData {
        variable: 0,
        bytes: [ 72, 101, 108, 108, 111, 32 ],
    },
    ToCioVecArray {
        variable_ptr: 1,
        variable_length: 2,
        arguments: [ 0 ],
    },
    CallPrint { arguments: [1, 2], },
]
```

And we can also check the WAT output, which is unchanged.

### SideQuest 2: Allowing many arguments

`print` in Ruby can actually receive many arguments but the current version of the compiler emits a call to `fd_write` for each argument. This can be seen by compiling the following ruby file
```ruby
print "Hello ", "World!\n"
```
which will generate the following WAT file
```wat
(module
  (import "wasi_snapshot_preview1" "fd_write" (func $print (param i32 i32 i32 i32) (result i32)))
  (memory (export "memory") 1)
  (data (i32.const 0) "Hello   \00\00\00\00\06\00\00\00World!\0a \10\00\00\00\07\00\00\00" )
  (func $_start (export "_start")
    i32.const 1
    i32.const 8
    i32.const 1
    i32.const 8
    call $print
    drop
    i32.const 1
    i32.const 24
    i32.const 1
    i32.const 24
    call $print
    drop
  )
)
```

In our current IR this is trivial to fix, when looping over the arguments we simply only push constants and then, at the end we do the conversion to a list of `ciovec` with all the constants added. The `print` parsing code is now
```rust
let before_varid = cur_varid;
for arg in &send.args {
    if let Str(raw_s) = arg {
        current_function.body.push(
            IR::StoreData {
                variable: cur_varid,
                bytes: Box::from(raw_s.value.as_raw().as_slice()),
            }
        );
        cur_varid += 1;
    } else return Err(/* */)
}

if before_varid != cur_varid {
    current_function.body.extend_from_slice(&[
        IR::ToCioVecArray {
            variable_ptr: cur_varid,
            variable_length: cur_varid + 1,
            arguments: (before_varid..cur_varid)
                .collect::<Vec<_>>()
                .into_boxed_slice(),
        },
        IR::CallPrint {
            arguments: [cur_varid, cur_varid + 1],
        },
    ]);
}
```

This gives the following WAT
```wat
(module
  (import "wasi_snapshot_preview1" "fd_write" (func $print (param i32 i32 i32 i32) (result i32)))
  (memory (export "memory") 1)
  (data (i32.const 0) "Hello World!\0a   \00\00\00\00\06\00\00\00\06\00\00\00\07\00\00\00" )
  (func $_start (export "_start")
    i32.const 1
    i32.const 16
    i32.const 2
    i32.const 16
    call $print
    drop
  )
)
```

Not perfect, as we could have created only one `ciovec`, but it is probably good enough for now

## Actually parsing variable assignments

Now we just need a symbol table and we are off to the races. We're using variable IDs all the way though at the moment anyway which means the symbol table can be as simple as a map of strings to ids. When the variable gets another assignment then we can just change the id in the table for that variable name. We also need anonymous variables so I'll roll that into the same structure for ease.

```rust
struct SymbolTable {
    current_id: usize,
    table: BTreeMap<String, usize>,
}

impl SymbolTable {
    fn new_anonymous(&mut self) -> usize {
        let id = self.current_id;
        self.current_id += 1;
        id
    }

    fn update_name(&mut self, name: String, variable: usize) {
        self.table.insert(name, variable);
    }

    fn get(&self, name: &String) -> Option<usize> {
        self.table.get(name).cloned()
    }
}
```

For ease I'm going to distinguish between lvalues and rvalues for now. Lvalues are anything to the left of and equals sign, and rvalues are anything to the right. We just need to make sure our main parsing loop is aware of assignments like so
```rust
Lvasgn(node) => {
    let var_id = symbol_table.new_named(node.name.to_owned());
    parse_rvalue(
        var_id,
        current_function,
        symbol_table,
    )?;
}
```

And then to parse those assignments we can do

```rust
fn parse_rvalue(
    this_varid: usize,
    current_function: &mut WasmFunction,
    symbol_table: &mut SymbolTable,
) -> Result<usize, RubyCompilerError> {
    use lib_ruby_parser::Node::*;
    match node {
        Str(raw_s) => {
            current_function.body.push(IR::StoreData {
                variable: this_varid,
                bytes: Box::from(raw_s.value.as_raw().as_slice()),
            });
            Ok(this_varid)
        }
        Lvar(x) => {
            if let Some(var) = symbol_table.get(&x.name) {
                Ok(var)
            } else { Err( /**/ ) }
        }
        // Skipped error handling
    }
}
```
This isn't the most beautiful code. I particularly dislike that we are reserving a new variable before parsing the rvalue, but I have been writing this post for about a month now, so this'll do.

We can also use this new rvalue parse for the arguments to print - so thats nice.

Finally we can attempt to use a variable
```ruby
x = "Hello"
print x, " World\n"
```

Which gives
```wat
(module
  (import "wasi_snapshot_preview1" "fd_write" (func $print (param i32 i32 i32 i32) (result i32)))
  (memory (export "memory") 1)
  (data (i32.const 0) "Hello World\0a\00\00\00\00\05\00\00\00\05\00\00\00\07\00\00\00" )
  (func $_start (export "_start")
    i32.const 1
    i32.const 12
    i32.const 2
    i32.const 12
    call $print
    drop
  )
)
```
💆. The code for the compiler at this point is available [here](https://github.com/xulaus/ruby2wasm/blob/640e333/src/main.rs).


---
[^1]:Cheating _ourselves_. The worst kind of cheating.
