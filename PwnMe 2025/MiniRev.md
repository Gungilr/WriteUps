
Description:
	A new and obscure programming language, MimiLang, has surfaced. It runs on a peculiar interpreter, but something about it feels… off. Dive into its inner workings and figure out what's really going on. Maybe you'll uncover something unexpected.

<h1>Get basic information </h1>

To preface this writeup, I enjoyed this challenge a lot. The solution was relatively simple but the details of the compiler surrounding the especially interesting. Looking at the description for the challenge reveals that the challenge is dealing with a rare coding language. Decompiling the program reveals that the script is a compiler for a custom programing language. Interesting here, is how GHIDRA and IDA handle the decomplication of the symbols differently. 

<p float="left">
  <img src="Pasted image 20250413023450.png" width="45%" />
  <img src="Pasted image 20250413023702.png" width="45%" />
</p>
For the purpose of this challenge using IDA seems more efficient as the symbols will be very helpful for the reversing process. Looking through the various folders and files we can quickly identify the general purpose of the program, which is to compile and run a file.  

Looking at the symbols names already reveal a lot of information about our target, however, before we look deeper into the disassembly let's start by running the file and looking at the output.

![[Pasted image 20250416000716.png]]

Looking at the output, more or less confirms our earlier speculation about the program being some sort of compiler. Let's create a empty program to pass into the program. 

![[Pasted image 20250416001451.png]]

Let's try a simple test file to avoid the empty file error:

![[Pasted image 20250416001646.png]]

These errors provide us crucial pieces of information to dig into in our next steps. Firstly it reveals the location of key  sections of the code, like the location of the parser, as well as a simple call stack. Another thing, is the fact that it's written in GO, which partially explains why my Ghidra was struggling earlier.

<h1> Digging into the Decompiler </h1>

At this point it's hard to get any more relevant information on the file without digging into the decompiled code. Although commands like `strings` or `objdump` might reveal additional information, we're gonna need to contextualize what we know first.  

Let's start with the main function since that's where the program starts:
![[Pasted image 20250416002327.png]]

The first things that is of immediate note is the various flags that our mini compiler takes. Things like the Debug mode and disassembly might be useful during the testing process.

![[Pasted image 20250416002709.png]]

Skimming past the rest of main we finally spot a few function calls. Some of them can just be dismissed as regular things done by GO_Lang like `runtime_slices` which is a function GO uses, but not relevant to the challenge. 

The line of interest to us in the image is the call function to `github_com_Lexterl33t_mimicompiler_compiler__ptr_Lifter_Compile`

Following the function calls shows us this set of instructions:
![[Pasted image 20250416003142.png]]

Remembering back to the parser location that we got from the error message, this is where its called. Looking for where this file is within the decompile we find a family of files that are interesting to us. The code is split into 2 sections, a vm section and and a compiler function, which suggests that the program is compiling the code then executing it.

![[Pasted image 20250416003429.png]]

This folder more or less contains all the compiler code we need to understand the syntax of the custom mimi language that we need to write. However, digging through all this code will take a while(I did it anyways because this was sick). So let's focus on what stands out. Immediately one function stands out to me, `VerifyProofStatrement`. Unlike the other functions, this function doesn't make sense within the compiler so let's take a closer look at it.

The compiler `VerifyProofStatrement` is simply the syntax breakdown, and the compiling the function, whereas the `VerifyProofStatrement` of the VM section has some very interesting information.
![[Pasted image 20250417174630.png]]
Seeing the mention of the flag in the function sets our goal to run this function properly. So let's break down how that works. Starting from the top of the function we see the the function pop 2 values off the stack, meaning it takes 2 variables as inputs.

![[Pasted image 20250417174900.png]]

Following that section of code we spot the key branch condition we need to fulfill to get to the flag.

![[Pasted image 20250417175330.png]]

The disassembly shows us the two conditions that need to be satisfied the mathematical function:
- First `param1 + param2 = 314159`
- Second there is a nonlinear function that's a applied to C that needs to be satisfied		
```c
x = a
y = b
v = y*y*y + x*x - x*y
v = ((v * MAGIC_CONST) >> 13) - (v >> 63)
v *= 0xFFFFD
x = x - v

assert(x == 0x42B6E)
```

This can be further simplified as the equation `y^3 + x^2 -xy % 1048573 == 273262`. We can tell this is a modulus in the original assembly by looking at `80001800...` and `0FFFFD` as magic bytes used to optimize modulus.

```python
def brute_force():

    target = 273262
    modulus = 1048573
    total = 314159

    for x in range(1, 314159):  # Reasonable range around known valid

        y = total - x

        r8 = x**2 + y**3 - x * y

        if r8 % modulus == target:

            print(f"Found valid (x, y): ({x}, {y})")

            return

  
brute_force()
```

Finding a solution for this set of conditions is relatively simple, given the limited number of combinations you can have that add to 314159 you can brute force a solution. Looking at the equations you can approximate that the numbers are roughly equal, I ran a brute force until I got the numbers `{123456, 190703}`. 


Calling the Verify Proof function with these numbers gets us the flag:
![[Pasted image 20250417185316.png]]