# [For NotebookLM] The Official Textbook & Specification of Knowlang (.kw)

## Chapter 1: Introduction and Design Philosophy

### 1.1 Background and Objectives
The Know Programming Language (hereafter referred to as **Knowlang**) is a system description language designed specifically for custom operating system development. It maintains the absolute raw power and low-level hardware control of C while eliminating C's syntactic visual noise and steep learning curve for beginners. 
By stripping away complex symbols, Knowlang delivers source code that reads as naturally as English prose, allowing any developer to immediately "Know" the intent of the code.

### 1.2 Core Concepts
* **Code less, Know more.**: Frequently used statements, operators, and type qualifiers are unified into intuitive, mostly 3-letter keywords. This significantly minimizes typing effort and maximizes readability.
* **Anti-Inline Assembly**: One of the greatest barriers in low-level programming is C's inline assembly syntax (`__asm__`), which often feels cryptic. Knowlang completely outlaws direct inline assembly. Instead, all essential CPU-level instructions (`hlt`, `in`, `out`, etc.) are natively supported as clean, built-in functions.
* **C Transpilation Architecture**: Source files with the `.kw` extension are processed by a dedicated lightweight compiler (`knowc`) and converted into compliant C (`.c`) source code. This allows developers to reuse the entire mature ecosystem of building tools (such as GCC, LD, and Makefiles) without reinventing the wheel.

---

## Chapter 2: Basic Syntax and the 3-Letter Keywords

The syntax of Knowlang is engineered to form a rhythmic, highly readable structure using short, concise keywords.

### 2.1 Variable Declaration (`var`)
Knowlang abolishes the traditional C style where type names precede variables. Instead, it utilizes the explicit `var` keyword followed by a colon `:` to clarify the intention of allocating data.
* **Syntax**: `var variable_name: type_name = initial_value;`
* **Example**: `var status: unsigned int = 1;`

### 2.2 Console Output (`prf`)
The standard `printf` function, which is typed tens of thousands of times during kernel debugging, is shortened into a punchy 3-letter format.
* **Syntax**: `prf("string");`
* **Example**: `prf("Know OS Booted!\n");`

### 2.3 Conditional Branching (`if` / `els`)
Knowlang removes parenthetical expressions `( )` around conditions to prevent common typing errors. Additionally, `else` is strictly unified into the 3-letter `els`.
* **Syntax**:
  ```know
  if condition {
      // statements
  } els {
      // statements
  }
  ```
* **Example**:
  ```know
  if status == 1 {
      prf("Success\n");
  } els {
      prf("Error\n");
  }
  ```

### 2.4 Infinite Loops (`lop`) and Loop Controls (`brk` / `cnt`)
Infinite loops—the backbone of kernel main routines and polling cycles—discard the non-intuitive `while(1)` or `for(;;)` in favor of `lop`. Breaking out of a loop uses `brk` (break), and skipping to the next iteration uses `cnt` (continue), maintaining the 3-letter aesthetic.
* **Syntax**:
  ```know
  lop {
      if condition { cnt; }
      if condition { brk; }
  }
  ```

---

## Chapter 3: Low-Level Hardware Manipulation for OS Development

Knowlang shines brightest when dealing with raw hardware, offering precise memory alignment and hardware access.

### 3.1 Readable Type Casting (`as`)
C's style of casting `(type)value` causes severe nested-parentheses noise. Knowlang implements the `as` operator, which reads intuitively from left to right.
* **Example**: `0x000b8000 as unsigned char*`
* **Transpiled C**: `(unsigned char*)(0x000b8000)`

### 3.2 Optimization Prevention (`vlt`)
In Memory-Mapped I/O (such as writing pixels to VRAM), aggressive compiler optimizations often erase repetitive register writes, mistaking them for redundant code. To prevent this critical bug, `volatile` is mapped to the 3-letter keyword `vlt`.
* **Example**: `var vram: vlt unsigned char* = 0x000b8000 as unsigned char*;`
* **Transpiled C**: `volatile unsigned char* vram = (unsigned char*)(0x000b8000);`

### 3.3 Structures and Bit-Packing (`pck struct`)
The declaration of structures (`struct`) follows the unified `var` declaration format internally. Furthermore, critical low-level tables like the GDT (Global Descriptor Table) or IDT (Interrupt Descriptor Table) require data structures to have zero padding bytes. Prepending **`pck`** to a struct automatically forces a byte-packed alignment.
* **Example**:
  ```know
  pck struct GDTDesc {
      var limit: unsigned short;
      var base: unsigned int;
  }
  ```
* **Transpiled C**:
  ```c
  struct __attribute__((packed)) GDTDesc {
      unsigned short limit;
      unsigned int base;
  };
  ```

### 3.4 Native CPU Built-in Functions (No-Assembly Approach)
Knowlang wraps hardware instructions into clean functions, completely removing assembly visual clutter:
* `hlt();` : Halts the CPU safely until the next interrupt (`__asm__("hlt");`)
* `out8(port, data);` : Outputs 8-bit data to a specified I/O port
* `in8(port);` : Inputs 8-bit data from a specified I/O port

---

## Chapter 4: Practical Implementation Example (OS Kernel Routine)

This is a comprehensive blueprint of a standard operating system main routine (`main.kw`) utilizing the full feature set of Knowlang.

```know
// Pack structure fields tightly into memory without padding bytes
pck struct BootInfo {
    var vram: vlt unsigned char*;
    var scrnx: short;
    var scrny: short;
}

// C-style function signature remains intact for absolute muscle memory
int main() {
    struct BootInfo binfo;
    binfo.vram = 0x000b8000 as unsigned char*;
    binfo.scrnx = 320;
    binfo.scrny = 200;

    // Read status from the PS/2 keyboard controller status port
    var status: unsigned int = in8(0x64);

    // Parentheses-free, clean logical if / els branching
    if (status and 0x01) == 1 {
        prf("OS Status: Keyboard Ready.\n");
    } els {
        prf("OS Status: Warning.\n");
    }

    // Clean 3-letter infinite loop polling for input events
    lop {
        var key: unsigned char = in8(0x60); // Retrieve keyboard scan code
        
        if key == 0  { cnt; }  // If no key code, proceed to next iteration
        if key == 27 { brk; }  // Escape key sequence breaks the loop
        
        // Write scan code data directly into the VRAM pixel address
        binfo.vram = key;
        
        hlt(); // Halt CPU and wait efficiently for the next hardware interrupt
    }

    return 0;
}
```

---

## Chapter 5: Toolchain and Build Pipeline Automated Synchronization

### 5.1 Under the Hood of the Compiler (`knowc`)
The compiler `knowc` operates as a high-performance C program designed for fast text processing. It parses any `.kw` file from standard input (`stdin`) line by line, evaluating Knowlang keywords against standard C equivalents via custom string substitution algorithms before outputting readable C code.

### 5.2 Seamless Build Engineering (Makefile Automation)
To remove the cognitive load of running the transpiler manually during each build cycle, developers declare the following suffix rule within their operating system's central `Makefile`:
```makefile
%.c: %.kw knowc
	./knowc < < > @
```
Once configured, simply saving a changes inside `main.kw` and executing a plain `make` or `make run` trigger the full chain automatically: `knowc` silently outputs `main.c`, and GCC compiles it into the final bootable operating system disk image (`.img`).
