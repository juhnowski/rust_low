# Исследование того, как Rust работает с памятью на низком уровне

```bash
cargo install cargo-show-asm
cargo build
cargo asm test1::main
```

# Статические переменные
```rust
static I: i32 = 1;

fn main() {
    println!("{}", I);
}
```

## Windows
```nasm
.section .text,"xr",one_only,test1::main,unique,4
	.globl	test1::main
	.p2align	4
test1::main:
	.cv_func_id 8
.seh_proc _ZN5test14main17h4452acf880167b3fE
	sub rsp, 56                     ; Выделяем 56 байт на стеке для локальных нужд
	.seh_stackalloc 56              ; Техническая разметка для Windows (обработка исключений)
	.seh_endprologue
	
	lea rax, [rip + test1::I]       ; Загружаем адрес переменной I в регистр RAX
	mov qword ptr [rsp + 40], rax   ; Сохраняем этот адрес на стек (ячейка [rsp + 40])
	lea rax, [rip + core::fmt::num::imp::<impl core::fmt::Display for i32>::fmt] ; Компилятор находит реализацию трейта Display для i32 типа <- динамическая диспетчеризация
	mov qword ptr [rsp + 48], rax   ; Кладется адрес функции форматирования на стек. 
	lea rcx, [rip + anon.127ef9f4afd56e9fba6304b17ce92b79.1]  ; В RCX адрес форматной строки "{}\n"
	lea rdx, [rsp + 40]                                       ; В RDX адрес массива аргументов (наш I и его Display)
	call std::io::stdio::_print                               ; вызов печати
	
	nop
	.seh_startepilogue
	add rsp, 56                                               ; Освобождаем стек
	.seh_endepilogue
	ret
```

## Linux
```asm
.section .text.test1::main,"ax",@progbits
	.hidden	test1::main
	.globl	test1::main
	.p2align	4
.type	test1::main,@function
test1::main:
	.cfi_startproc
	sub rsp, 24
	.cfi_def_cfa_offset 32
	lea rax, [rip + test1::I]
	mov qword ptr [rsp + 8], rax
	mov rax, qword ptr [rip + core::fmt::num::imp::<impl core::fmt::Display for i32>::fmt@GOTPCREL]
	mov qword ptr [rsp + 16], rax
	lea rdi, [rip + .Lanon.8093c3e359ed4215fb983c74819fad58.1]
	lea rsi, [rsp + 8]
	call qword ptr [rip + std::io::stdio::_print@GOTPCREL]
	add rsp, 24
	.cfi_def_cfa_offset 8
	ret
	```

# Размещение на куче
```rust
let heap = Box::new(3);
println!("{}", heap);
```

## Linux
```asm
.section .text.test1::main,"ax",@progbits
	.hidden	test1::main
	.globl	test1::main
	.p2align	4
.type	test1::main,@function
test1::main:
	.cfi_startproc
	.cfi_personality 155, DW.ref.rust_eh_personality
	.cfi_lsda 27, .Lexception0
	push rbx
	.cfi_def_cfa_offset 16
	sub rsp, 32
	.cfi_def_cfa_offset 48
	.cfi_offset rbx, -16
	call qword ptr [rip + __rustc::__rust_no_alloc_shim_is_unstable_v2@GOTPCREL]
	
	mov edi, 4  ; size, 4 байта для i32
	mov esi, 4  ; align, выравнивание 4 байта
	call qword ptr [rip + __rustc::__rust_alloc@GOTPCREL] ; выделяем память в куче
	test rax, rax ; проверяем, успешно ли выделена память
	je .LBB4_4  ; если память не выделена, переходим к обработке ошибки
	mov dword ptr [rax], 3  ; записываем значение 3 в выделенную память
	mov qword ptr [rsp + 8], rax ; сохраняем указатель на выделенную память в куче на стек
	lea rax, [rsp + 8]  ; загружаем указатель на 3 в куче из стека в rax
	mov qword ptr [rsp + 16], rax ; сохраняем указатель в стеке
	lea rax, [rip + <alloc::boxed::Box<T,A> as core::fmt::Display>::fmt] ; загружаем указатель на функцию fmt в rax для трейта Display для типа Box
	mov qword ptr [rsp + 24], rax
	lea rdi, [rip + .Lanon.8093c3e359ed4215fb983c74819fad58.1]
	lea rsi, [rsp + 16]
	call qword ptr [rip + std::io::stdio::_print@GOTPCREL] ; печатаем
	mov rdi, qword ptr [rsp + 8]  ; достаем адрес из кучи
	mov esi, 4 ; size, 4 байта для i32
	mov edx, 4 ; align, выравнивание 4 байта
	call qword ptr [rip + __rustc::__rust_dealloc@GOTPCREL] ; освобождаем память в куче
	add rsp, 32 ; чистим стек
	.cfi_def_cfa_offset 16
	pop rbx
	.cfi_def_cfa_offset 8
	ret
.LBB4_4:
	.cfi_def_cfa_offset 48
	mov edi, 4
	mov esi, 4
	call qword ptr [rip + alloc::alloc::handle_alloc_error@GOTPCREL]
	mov rbx, rax
	mov rdi, qword ptr [rsp + 8]
	mov esi, 4
	mov edx, 4
	call qword ptr [rip + __rustc::__rust_dealloc@GOTPCREL]
	mov rdi, rbx
	call _Unwind_Resume@PLT
	```
