# Исследование того, как Rust работает с памятью на низком уровне

```bash
cargo install cargo-show-asm
cargo build
cargo asm test1::main
```

# Windows
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

# Linux
