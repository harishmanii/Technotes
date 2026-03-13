# Linux Program Execution – binfmt_misc, Scripts, and Interpreters

## 1. `/proc/sys/fs/binfmt_misc`

`binfmt_misc` is a Linux kernel feature that allows registering **new executable binary formats**.

It acts as an **extension mechanism for the `execve()` system call**, allowing the kernel to execute files that are not native Linux binaries.

Location:

```

/proc/sys/fs/binfmt_misc/

```

Typical entries:

```

register
status
qemu-arm
qemu-aarch64

```

Each entry represents a **binary format handler**.

Example rule concept:

```

If file matches pattern X → execute interpreter Y

```

Example:

```

.exe → wine
.jar → java -jar
ARM binary → qemu-arm

```

This mechanism is commonly used for:

- Running foreign CPU binaries (QEMU)
- Running Windows programs (Wine)
- Multi-architecture Docker containers
- Custom executable formats

---

# 2. How Linux Executes Programs

When a user runs a program:

```

./program

```

The shell performs:

```

execve("./program")

```

Inside the kernel, Linux determines **how to execute the file** using binary format handlers.

Simplified execution order:

```

execve()
↓
search_binary_handler()
↓

1. binfmt_elf     → ELF executables
2. binfmt_script  → scripts with #!
3. binfmt_misc    → registered custom formats

```

---

# 3. ELF Executables

ELF (Executable and Linkable Format) is the **native executable format used by Linux**.

Example:

```

/bin/ls
/usr/bin/python3
/bin/bash
/usr/bin/java

```

When executing an ELF file:

```

User
↓
execve()
↓
Kernel loads ELF
↓
Program starts executing

```

The kernel loads program segments into memory such as:

```

Code (.text)
Data (.data)
Heap
Stack

````

---

# 4. Script Execution (`#!` Shebang)

Scripts are handled by the **script binary handler (`binfmt_script`)**.

Example script:

```python
#!/usr/bin/python3
print("hello")
````

When running:

```
./script.py
```

Kernel behavior:

```
execve("./script.py")
      ↓
kernel reads first two bytes
      ↓
#! detected
      ↓
interpreter extracted
```

Kernel internally executes:

```
execve("/usr/bin/python3", ["python3", "./script.py"])
```

So the actual program executed is:

```
/usr/bin/python3  (ELF binary)
```

---

# 5. Interpreted Languages

For interpreted languages, the kernel executes the **interpreter ELF binary**, not the source code.

General architecture:

```
Kernel
  ↓
ELF Interpreter
  ↓
User program
```

Examples:

| Language | Interpreter ELF  |
| -------- | ---------------- |
| Python   | /usr/bin/python3 |
| Bash     | /bin/bash        |
| PHP      | /usr/bin/php     |
| NodeJS   | /usr/bin/node    |

Example execution flow:

```
./script.py
   ↓
kernel detects #!
   ↓
executes python3 ELF
   ↓
python interpreter reads script
   ↓
script executes
```

---

# 6. Virtual Machine Languages (Example: Java)

Java uses a **virtual machine runtime (JVM)**.

Example:

```
java Hello.class
```

Execution flow:

```
Kernel
  ↓
execve("/usr/bin/java")
  ↓
kernel loads JVM ELF
  ↓
JVM starts
  ↓
JVM loads bytecode (.class)
  ↓
JVM executes program
```

The kernel never executes `.class` files directly.

---

# 7. Compiled Languages

Compiled languages produce **ELF executables directly**.

Example:

```
gcc hello.c
```

Output:

```
hello  → ELF binary
```

Execution flow:

```
User
 ↓
execve("./hello")
 ↓
Kernel loads ELF
 ↓
Program runs directly
```

No interpreter is involved.

---

# 8. Execution Model Summary

| Program Type | Example       | Kernel Executes |
| ------------ | ------------- | --------------- |
| Compiled     | C program     | Program ELF     |
| Interpreted  | Python script | Python ELF      |
| VM-based     | Java          | JVM ELF         |

Key rule:

> The Linux kernel executes **ELF binaries**, not programming language code.

---

# 9. Mental Model

```
User Program
     ↓
Interpreter / Runtime (ELF)
     ↓
Kernel loads ELF
     ↓
Program execution
```

Examples:

```
script.py  → python3 ELF
test.sh    → bash ELF
Hello.class → java ELF
program.c  → compiled ELF
```

---

```