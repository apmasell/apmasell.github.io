# Using a Custom Compiler to Reduce Toil
At OICR, we analyse samples prepared in our in-house lab. The laboratory staff enters data into [MISO](https://miso-lims.readthedocs.io/projects/docs/en/latest/), our laboratory information system, about each sample and how it was processed. What analysis we need to do depends on this information. That metadata is...not perfect. Sometimes there are data entry errors and some times additional information is added as the lab does quality control based on the analysis results. Sometimes there are changes based on lab housekeeping (e.g., transfer of materials between departments, re-labelling).

To get around this, we have an abstraction layer for data from the LIMS, called [Pinery](https://github.com/oicr-gsi/pinery), that computes a hash of the metadata. If information changes in MISO, the Pinery hash will change and we know that the analysis may be incorrect.

This turns out to be too conservative though. For instance, if the lab moves a sample to a different freezer, this changes the metadata, but in a way that almost no analysis cares about. We can't simply exclude this field because some of that information does make it into QC reports. Unfortunately, a human has to reconcile these records and decide whether to re-analyse the data or update the hash. Determining which pieces of metadata are important is hard, differs for different analysis types, and it is easy to make a mistake.

Fortunately, the [Shesmu](https://github.com/oicr-gsi/shesmu) compiler can come to the rescue. Shesmu extracts all kinds of metadata from Pinery and makes it available to a small programs, called an _olive_, that direct what analysis to do. Here's an extract from an olive that runs BWAmem:

```
Where
    metatype =="chemical/seq-na-fastq-gzip"
    && (kit == "10X Genome Linked Reads") == linked_reads
    && workflow In ["bcl2fastq", "CASAVA", "chromium_mkfastq", "FileImportForAnalysis"]
    && library_design In ["WG", "EX" ,"TS", "NN", "CH", "AS", "CM"]
    && tissue_type != "X"
    && config::project_info::get(project).cancer_pipeline != ``
```

In this example, `kit`, `library_design`, `tissue_type`, and `project` come from Pinery. Other values, including, `metatype` and `workflow`, come from other sources. Since Shesmu knows that these values are important, it can use them to create a signature of only the data that is important.

The author of an input format can specify what fields are signable and the Shesmu compiler will track all signable variables. There are then signers, which are algorithms to convert these variables into an output signature. Built-in to Shesmu are:

- `std::signature::sha1` - create a stable SHA1 hash from all the signable variables
- `std::json::signature` - create a JSON object with all of the signable variables as fields

Additional signers can be added to Shesmu using its plugin system.

Since a signature uses only the data we know was required. This makes it possible to automatically determine if the changes in MISO should trigger re-analysis or not. If the hashes from Pinery are mismatched, but the Shesmu signature match, the Pinery signatures of the analysis can be automatically updated. If the Shesmu signatures do not match, but the Pinery hashes match, a structural change has been made to the olive and it is equivalent to the old analysis, so the Shesmu signature can be updated. If both mismatch, then re-analysis is required.

This drastically reduces the amount of toil our operations group has to do sorting these out. The additional cost to the olive developers to use the signatures is tiny and requires to updating as the olive is changed and different fields are used.
