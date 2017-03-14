OSDI Lab 2
===================
1.1. Add image before system start
-------------
#### 1. Use qemu option
```{r, engine='bash', code_block_name}
-boot menu=on
-boot splash-time=1000
-boot splash=/path/to/bootsplash.png
```
#### 2. Use coreboot & seabios
```{r, engine='bash', code_block_name}
git clone git://git.seabios.org/seabios.git seabios
cd seabios
make menuconfig
make
```
Setting Build target = coreboot, Select Bootmenu 
```{r, engine='bash', code_block_name}
git clone https://review.coreboot.org/coreboot.git
cd coreboot
make menuconfig
make
```
Setting bootsplash = bootsplash.jpg, payload = bios.bin.elf
```{r, engine='bash', code_block_name}
qemu-system-i386 -bios coreboot.rom ...
```
2.3. Multibooting support
-------------
linux-0.11/Makefile
```
Image: boot/bootsect boot/setup tools/system boot/hello
 	@cp -f tools/system system.tmp
 	@strip system.tmp
 	@objcopy -O binary -R .note -R .comment system.tmp tools/kernel
	@tools/build.sh boot/bootsect boot/setup tools/kernel Image boot/hello $(ROOT_DEV)
 	@rm system.tmp
 	@rm tools/kernel -f
 	@sync
 
boot/hello: boot/hello.s
	@make hello -C boot
```
boot/Makefile
```
all: bootsect setup hello

hello: hello.s
	@$(AS) -o hello.o hello.s
	@$(LD) $(LDFLAGS) -o hello hello.o
	@objcopy -R .pdr -R .comment -R.note -S -O binary hello
	
@rm -f bootsect bootsect.o setup setup.o head.o hello hello.o
```
boot/bootsect.s
```
.equ HELLOLEN, 1		# nr of hello-sectors
.equ HELLOSEG, 0x9920		# setup starts here
# press 1 or 2
keyboard:
	mov     $0x0000, %AH             # function number = 00h :
	int     $0x16                   # call INT 16h, BIOS Keyboard services
	cmp     $0x31, %AL
	je      load_setup
	cmp     $0x32, %AL
	je      load_hello
	jmp     keyboard

load_hello:
	mov	$HELLOSEG, %ax
	mov	%ax, %es
 	mov	$0x0000, %dx		# drive 0, head 0
 	mov	$0x0002, %cx		# sector 2, track 0
	mov	$0x0000, %bx		# address = 512, in INITSEG
	.equ    AX, 0x0200+HELLOLEN
	mov     $AX, %ax		# service 2, nr of sectors
	int	$0x13			# read it
	jnc	ok_load_hello		# ok - continue
	mov	$0x0000, %dx
	mov	$0x0000, %ax		# reset the diskette
	int	$0x13
	jmp	load_hello

ok_load_hello:
	ljmp	$HELLOSEG, $0

load_setup:
	mov	$0x0000, %dx		# drive 0, head 0
	mov	$0x0003, %cx		# sector 2, track 0
```
tools.build.sh
```
hello_img=$5
root_dev=$6

# Write hello(512bytes, four sectors) to stdout
[ ! -f "$hello_img" ] && echo "there is no hello binary file there" && exit -1 
dd if=$hello_img seek=1 bs=512 count=1 of=$IMAGE 2>&1 >/dev/null

# Write setup(4 * 512bytes, four sectors) to stdout
 [ ! -f "$setup" ] && echo "there is no setup binary file there" && exit -1
dd if=$setup seek=2 bs=512 count=4 of=$IMAGE 2>&1 >/dev/null

dd if=$system seek=6 bs=512 count=$((2888-1-4)) of=$IMAGE 2>&1 >/dev/null
```
