# Getting it done fast and fast with Rust
At OICR, I was writing a workflow for bcl2fastq, the tool that extracts data from Illumina sequencers into FASTQ, a standard text-based format describing DNA sequences. Our sample preparation team was testing a new technique called unique molecular identifiers (UMIs) that allow determining if duplicate sequences in the output are real biological duplication or artifacts of the sample preparation process.

We decided that integrating the UMI processing with the bcl2fastq was desirable since the UMI processing also produces a FASTQ that is almost identical to the original and having two large duplicate FASTQs was a waste of storage.

A co-worker developed a program called [Barcodex](https://github.com/oicr-gsi/barcodex) in Python to do the UMI extraction. I modified our bcl2fastq workflow to accommodate it and...it was slow. Pushing a multi-gigabyte text file through Python turned out to not meet our performance requirements. The deadline for this project was yesterday, so I needed a way to improve performance and get it done fast.

I decided to abandon the Python program and rewrite it in Rust, resulting in [Barcodex-RS](https://github.com/oicr-gsi/barcodex-rs). It took me 10 hours to read the Python, port it to Rust, test it on its own, and then integrate it into the bcl2fastq workflow. The debugging was less than half that time and I encountered only one non-trivial bug. The Rust version ran in a little under 2 hours on data that had taken the Python version over 24 hours.

I think it speaks to Rust's design that I could port a program in under two days and get a 10× performance improvement. It's worth noting that I made *no* algorithmic improvements to this software; all of the performance gains have come purely from the overhead Rust was able to remove.

While I'm sure I could have gotten similar performance using C or C++, I strongly doubt I could have done so in the same development time. I spent no time debugging memory errors or having to validate my program through Valgrind. Most of my debugging was trivial errors (inverted logic conditions, incorrect case handling, string indexing mistakes) and one was a problem with my library (the output from bcl2fastq is gzipped using a multi-session file and the gzip library I was using was expecting a single session file).

While Barcodex-RS was not my first Rust program, it was my first Rust program used in production. It was a test of how quickly I could get Rust to do what I needed and it really exceeded my expectations. I would strongly advocate for using it again in a situation where performance *and* short development time is needed.
