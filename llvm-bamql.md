# Using LLVM for fun and performance

BAM is a common biological data format composed of reads, individual sequences of DNA observed using a DNA sequencing instrument and the context where they belong in an organism's genome. Pulling out a subset of reads is a common activity and usually accomplished by converting the file to SAM, a text equivalent to BAM, manipulating the results using grep and awk, and then repacking the output into BAM.

To make this a less painful task, I developed [BAMQL](https://github.com/BoutrosLaboratory/bamql). It has a small domain-specific programming language with predicates that can ask questions about a BAM reads. It can compile the query into assembly language using LLVM and execute it using LLVM's built-in JIT. It uses [htslib](https://github.com/samtools/htslib), the official C library for reading BAM files, plus a small core of C and C++ code to filter BAM files in various ways. A query can look something like this:

```
chr(1) & read_group ~ /C3BUK.1/ | chr(2)
```

The compiler uses a simple recursive-descent parser written in C++ to parse and check the query. The language is intentionally very simple, so there's almost no checking to be done. In fact, parsing, name resolution, and type checking are all done as a single operation. The `chr` predicate checks what read a chromosome is on. Because standards are hard, some BAM files label chromosome one as simply `1` while others use `chr1`; BAMQL checks both for you. It has knowledge of other synonyms for chromosomes, such as `23` and `X`, or `24` and `Y`.

Performance tests show that BAMQL is almost as fast as hand-written C code and faster than using htslib in Python or PERL, which both have wrappers.

For the query:

```
foo = chr(1) & paired?;
```

this is the LLVM bitcode generated:

```
define zeroext i1 @foo(%struct.bam_hdr_t* %header, %struct.bam1_t* %read, void (i8*, i8*)* %error_fn, i8* %error_ctx) {
entry:
  %0 = load i8*, i8** @.regex
  %1 = call i1 @bamql_check_chromosome(%struct.bam_hdr_t* %header, %struct.bam1_t* %read, i8* %0, i1 false)
  %2 = icmp eq i1 %1, false
  br i1 %2, label %merge, label %next


merge:                                            ; preds = %next1, %next, %entry
  %3 = phi i1 [ %1, %entry ], [ %6, %next ], [ true, %next1 ]
  ret i1 %3


next:                                             ; preds = %entry
  %4 = call i32 @bamql_flags(%struct.bam1_t* %read)
  %5 = and i32 %4, 1
  %6 = icmp eq i32 %5, 1
  %7 = icmp eq i1 %6, false
  br i1 %7, label %merge, label %next1


next1:                                            ; preds = %next
  br label %merge
}
```

The functions `bamql_check_chromosome` and `bamql_flags` are included as part of a small [support library](https://github.com/BoutrosLaboratory/bamql/blob/master/runtime/runtime.c) included with BAMQL that mostly acts as a wrapper around htslib. Some part of htslib are implemented as macros, so the support library provides real functions LLVM can use. Although the support library functions are C, LLVM allows adding extra information to the definitions in order to preform optimisation. Because I know their exact behaviour, I can specify restrictions on them that the C compiler would not, such as the functions being pure, not altering memory, or performing I/O, allowing LLVM to generate more optimised code in some situations.

Regular expressions are available to users and are implemented using PCRE. To improve performance, they are compiled (using PCRE's regular expression compiler, not LLVM). Since C++ has constants that are objects, LLVM has a way to declare initialization blocks. That is where the regular expressions get compiled:

```
@.regex = private global i8* null
@.str = private constant [10 x i8] c"^(chr)?1$\00", align 1
@.regex.1 = private global i8* null
@llvm.global_ctors = appending global [1 x %0] [%0 { i32 65535, void ()* @__ctor, i8* null }]
@llvm.global_dtors = appending global [1 x %0] [%0 { i32 65535, void ()* @__dtor, i8* null }]

define internal void @__ctor() {
entry:
  %0 = call i8* @bamql_re_compile(i8* getelementptr inbounds ([10 x i8], [10 x i8]* @.str, i8 0, i8 0), i32 4097, i32 0)
  store i8* %0, i8** @.regex
  %1 = call i8* @bamql_re_compile(i8* getelementptr inbounds ([10 x i8], [10 x i8]* @.str, i8 0, i8 0), i32 4097, i32 0)
  store i8* %1, i8** @.regex.1
  ret void
}


define internal void @__dtor() {
entry:
  call void @bamql_re_free(i8** @.regex)
  call void @bamql_re_free(i8** @.regex.1)
  ret void
}
```

The `__ctor` function calls the regular expression compiler with the strings baked into the assembler and puts them into a global variable used later. The `__dtor` function does the matching clean-up, ensuring that no memory is leaked when the application exits. The `llvm.global_ctors` and `llvm.global_dtors` contain a list of all constructors and destructors, respectively, and a number that determines initialisation order. This complexity is intended for C++ where many blocks will be created in different compilation units.

BAMQL can even produce debugging symbols. If one of these queries is compiled into object code and linked into a larger C or C++ program (the bamql-compile program will emit object code and header files), then a standard debugger can include the correct source of the BAMQL query.
