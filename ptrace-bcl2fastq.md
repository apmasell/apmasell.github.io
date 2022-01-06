# Don't argue with me, I know `ptrace`

At OICR, we use Illumina DNA sequencing instruments which produce more output than we would want for a single sample. We, like most Illumina users, multiplex multiple samples by attaching identifying bits of DNA called barcodes or indices. We can then demultiplex using these barcodes and recover our individual samples via the Illumina `bcl2fastq` tool.

This workflow has a frustrating problem: it is extremely batch oriented. If we discover a mistake in the barcode assignments, we have to re-run `bcl2fastq` and re-process *everything* that was sequenced together. Illumina's technology is trending toward large and larger batches. Re-processing more than a single troubled sample causes delays and extra work for other samples that happen to be sequenced at the same time.

The obvious solution is to run the tool once for each sample and throw away the other data. This was our plan too, but `bcl2fastq` proved to be...uncooperative. Any sequences that are not matched to the sample go into an _undetermined_ file. Running a single sample at a time means that the undetermined file would be almost all of the data produced, which can be hundreds of gigabytes. We often do runs with over 90 samples, so having a terabyte of disk usage, even if it's temporary, for data we have no intention to use is a bit of a show stopper.

The Illumina tool does not have a way to turn this undetermined output off and the tool is closed source. However, I was able to add such a feature using Linux's `ptrace` system.

You may have used the `ptrace` utility on the command line which traces every system call a program makes. A system call is when the program makes a request to the Linux kernel, which includes managing child programs, managing threads, and all input/output operations. The `ptrace` system allows another program to intercept system calls. Often, this is used for debugging purposes, but it can be adapted for more...nefarious purposes.

I built a tool which uses `ptrace` to intercept system calls in `bcl2fastq` and change requests to open the undetermined output file to instead open `/dev/null`, causing all the undetermined data to go nowhere.

Conceptually, the tool, called [`bcl2fastq-jail`](https://github.com/oicr-gsi/bcl2fastq/blob/master/wrapper/main.cpp), works like this:

- it pre-processes some input for `bcl2fastq` (this is not required for the tracing, but part of our overall design)
- it then splits into two processes using the `fork` system call
- the child declares that it is willing to be traced and stops itself
- the parent starts the tracing process and restarts the child
- the child executes `bcl2fastq`
- the parent intercepts system calls doing rewriting as described below
- once the child exits, the parent performs some post-processing (again, not required by the tracing, but part of our overall design)

There are two pairs of system calls we intercept:

`open` and `openat` - these open a file. Fundamentally, they are similar, but make different assumptions about how relative file paths are handled
`stat` and `lstat` - these get information about a file, including its size


We don't want to interfere with every system call, so the code works like this:

- check if the system call is one of the above; if it isn't, continue as normal
- check if the file name for one of the system calls is the _undetermined_ file; if it isn't, continue as normal
- if the program is trying to open a file with `Undetermined` in the name, modify the process's memory so it is open `/dev/null` instead
- if the program is trying to stat `/dev/null`, change the file name to `/bin/ls`

So, bcl2fastq will start and open the files it needs with the undetermined data sneakily going nowhere. It then does its normal processing activities. At the end of its run, it creates a statistics file that includes the sizes of the files it has written. To get that information, it has to stat the output files. When it tries to stat `/dev/null` it does not get back a file size because `/dev/null` is a device file and promptly freaks out. To combat this, the stat call is redirected to `/bin/ls` which is a regular file. This means the size for the undermined file will be incorrect, but since we already decided we didn't care about it, it's no big loss. This requires rewriting memory inside `bcl2fastq` and care has to be taken to ensure that the replacement strings were smaller than the original strings and appropriate nul-termination is done. It is fortunate that Undetermined has 12 bytes and `/dev/null` has 9 bytes. I chose to use `/bin/ls` since I knew it could fit into the available memory. To complicate matters, `ptrace` allows reading or writing client memory only in long-sized chunks, so care and nul-padding has to be taken to work in 8 byte chunks. Although the program looks for the sequence `Undetermined`, the file name used by `bcl2fastq` will also include an extension, so it is guaranteed that there will be 16 bytes we can overwrite. This causes slight output corruption: `bcl2fastq` is a C++ program and uses `std::basic_string`, which is a fixed length string, while the kernel uses nul-terminated C strings. Once the file name has been over written, the kernel will see the exact string we want, but C++ program will see the additional nul bytes and put them into its log. This is a minor inconvenience and we do not use the logs anyway.

Since `bcl2fastq-jail` has to validate every system call, there is a performance cost to this approach. In practice, the overhead introduced is small enough that we don't notice. In general, `bcl2fastq` is a heavily I/O-bound workload, so adding some CPU cost doesn't degrade performance significantly.

This method has been working reliably in production for a year. There have been a few pains where `bcl2fastq-jail` did not handle UNIX signals correctly. `ptrace` notifies the parent that a system calls are happening in the target using signals and determining when a signal should be trapped by the parent and when the signal should be handled by the child is confusing. `bcl2fastq` is written in C++ and a number of exceptions can be thrown that raises `SIGABORT` and requires the jail to crash appropriately.

Overall, this has been a successful method for dealing with uncooperative closed-source software. Certainly, this is a relatively easy case since the operations we needed to modify mapped on to system calls. It would be much more difficult to intervene if we wanted to change what data was written to the output files.
