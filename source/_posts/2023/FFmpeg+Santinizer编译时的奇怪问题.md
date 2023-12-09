---
title: FFmpeg+Santinizer编译时的奇怪问题
date: 2023/12/05 11:11:12
categories:
- Hack
tags:
- Fuzz
---

FFmpeg+Santinizer编译时的奇怪问题，不得不感叹软件世界的复杂性，总是会有奇奇怪怪的问题，还好有开源，不然都无法debug。

<!-- more -->

fuzzbench里编译ffmpeg的时候出现了奇怪的编译报错（剧透：和sanitizer有关系），首先上来就说我符号找不到，而且很多。搜了一下发现大多都是汇编里的符号，ffmpeg里面很多手写的汇编代码，直接和C语言链接起来。

```
ld.lld: error: undefined symbol: ff_yuv2yuv_422p10to8_sse2
>>> referenced by colorspacedsp_init.c:96 (/src/ffmpeg/libavfilter/x86/colorspacedsp_init.c:96)
>>>               colorspacedsp_init.o:(ff_colorspacedsp_x86_init) in archive libavfilter/libavfilter.a

ld.lld: error: undefined symbol: ff_yuv2yuv_422p10to10_sse2
>>> referenced by colorspacedsp_init.c:96 (/src/ffmpeg/libavfilter/x86/colorspacedsp_init.c:96)
>>>               colorspacedsp_init.o:(ff_colorspacedsp_x86_init) in archive libavfilter/libavfilter.a

ld.lld: error: undefined symbol: ff_yuv2yuv_422p10to12_sse2
>>> referenced by colorspacedsp_init.c:96 (/src/ffmpeg/libavfilter/x86/colorspacedsp_init.c:96)
>>>               colorspacedsp_init.o:(ff_colorspacedsp_x86_init) in archive libavfilter/libavfilter.a

ld.lld: error: undefined symbol: ff_yuv2yuv_422p12to8_sse2
>>> referenced by colorspacedsp_init.c:96 (/src/ffmpeg/libavfilter/x86/colorspacedsp_init.c:96)
>>>               colorspacedsp_init.o:(ff_colorspacedsp_x86_init) in archive libavfilter/libavfilter.a

ld.lld: error: too many errors emitted, stopping now (use --error-limit=0 to see all errors)
clang++: error: linker command failed with exit code 1 (use -v to see invocation)
```

辛酸的debug过程就不说了。只能说还是得多搜代码，同时找一个正确编译的环境去对比。

最后发现问题在于configure的时候，makefile里多了一个`X86ASMFLAGS=-f elf64 -DPIC -DPREFIX -g -F dwarf`。关键在于这个`-DPREFIX`，它会导致汇编链接出来的符号多一个下划线前缀，从而导致符号找不到。

而为什么会增加`-DPREFIX`这个配置？发现是在configure.sh里由`extern_prefix`变量控制的。这个变量的判断规则如下：让编译器创建一个只有一个全局变量的对象，然后用nm读取符号名字，看看有没有什么前缀。

```
test_cc <<EOF || die "Symbol mangling check failed."
int ff_extern;
EOF
sym=$($nm $TMPO | awk '/ff_extern/{ print substr($0, match($0, /[^ \t]*ff_extern/)) }')
extern_prefix=${sym%%ff_extern*}
```

**有问题的构建**

```
root@2c0808a63794:/src/ffmpeg# nm -g /tmp/ffconf.0T9GnOK3/test.o
0000000000000008 C ___asan_globals_registered
                 U __asan_init
                 U __asan_register_elf_globals
                 U __asan_unregister_elf_globals
                 U __asan_version_mismatch_check_v8
0000000000000000 B __odr_asan_gen_ff_extern
                 w __start_asan_globals
                 w __stop_asan_globals
0000000000000000 B ff_extern
```

**正常的构建**

```
root@c043e1ed694c:/src# nm -g /tmp/ffconf.OTqSYv4r/test.o
                 U __asan_init
                 U __asan_register_globals
                 U __asan_unregister_globals
                 U __asan_version_mismatch_check_v8
0000000000000000 B ff_extern
```

### 原因

可以发现，这个awk的匹配可能有问题，匹配到了__odr_asan_gen_ff_extern，认为会有前缀，且前缀是`__odr_asan_gen_`。然而其实根本没有前缀，那个符号就好好地在最下面。导致了ffmpeg对配置的误判。

但是这个`__odr_asan_gen_`前缀是什么呢？搜了一下发现了大佬的[博客](https://maskray.me/blog/2023-10-15-address-sanitizer-global-variable-instrumentation#odr-indicator)。ODR代表One Definition Rule，一个名字只能对应一个定义的变量，这里sanitizer尝试帮你判断了是否有重名的变量，然后这个符号就是这个功能产生的。关闭odr相关功能可能就可以解决这个问题？

可能就是因为用的clang版本太新了，新版的把`-fsanitize-address-use-odr-indicator`设为了默认。这里尝试把它关掉，即加上`-fno-sanitize-address-use-odr-indicator`。
