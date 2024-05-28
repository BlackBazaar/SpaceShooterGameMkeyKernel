- For run this project at kernel, after get into the file with cd Desktop, cd MkeyProject

- You should run these codes:

nasm -f elf32 kernel.asm -o kasm.o
gcc -fno-stack-protector -m32 -c kernel.c -o kc.o
ld -m elf_i386 -T link.ld -o kernel kasm.o kc.o
qemu-system-i386 -kernel kernel


- kernel.c is the main file inside of this project, i just put it for showing the main codes.