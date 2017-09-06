<h1 align="center">
    <i>lima</i> - The PacBio Barcode Demultiplexer
</h1>

<p align="center">
  <img src="img/lima.png" alt="Logo of Lima" width="150px"/>
</p>

## Scope
*Lima*, the PacBio read demultiplexer, is the standard tool to identify
barcode sequences in PacBio single-molecule sequencing data and is the successor
to `pbbarcode` and `bam2bam` for demultiplexing, starting with SMRT Analysis v5.1.0.
This new tool provides better user experience.

## Availability
The latest pre-release linux binary can be found under
[releases](https://github.com/PacificBiosciences/barcoding/releases).

Official support is only provided for official and stable SMRT Analysis builds
provided by PacBio.

Unofficial support for binary pre-releases is provided via github issues.

## Background
*Lima* can demultiplex samples that have a unique per-sample barcode pair and
have been pooled and sequenced on the same SMRT cell. Aside from the different
techniques how to associate the barcodes with the sample, by PCR or ligation,
there are three different barcode library designs.
In order to describe a barcode library design, one can view it from a
SMRTBell or read perspective. As *lima* supports raw subread and ccs read
demultiplexing, the following terminology is based on the the per (sub-)read view.

<img src="img/barcode_overview.png" width="1000px">

In the overview above, the input sequence is flanked by adapters on both sides.
The bases adjacent to an adapter are referred to as barcode regions.
A read can have up to two barcode regions, leading and trailing.
Either or both adapters can be missing and consequently the leading and/or
trailing region is not being called.

For the symmetric and tailed library design, the *same* barcode is attached to
both sides of the insert sequence of interest; the only difference is the
orientation of the trailing barcode. For identification, one read with a single
barcode region is sufficient.

For the asymmetric design, a *different* barcode pair is attached to the sides
of the insert sequence of interest. In order to be able to identify a
*different* barcode pair, a read with leading and trailing barcode regions
is required.

Output barcode pairs are generated from the identified barcodes.
The barcode names are combined using the `--` infix, for example `bc1002--bc1054`.
The sort order is defined by the barcode indices, lowest first.

## Features

*Lima* offers following features:
 * Process both, raw subreads and ccs reads
 * BAM in- and output
 * Extensive reports that allow in-depth quality control
 * Clip barcode sequences and annotate `bq` and `bc` tags
 * Agnostic of input barcode sequence orientation
 * Split output by barcode
 * No scraps.bam needed
 * Full PacBio dataset support
 * Peek into the first N ZMWs and get average barcode score
 * Guess set of barcodes given a mean barcode score threshold
 * Enhanced filtering options to remove ambiguous calls

## Execution
**Attention: Existing output files will be overwritten!**

Run on raw subread data:

    lima movie.subreads.bam barcodes.fasta prefix.bam
    lima movie.subreadset.xml barcodes.barcodeset.xml prefix.subreadset.xml

Run on CCS data:

    lima --css movie.ccs.bam barcodes.fasta prefix.bam
    lima --ccs movie.consensusreadset.xml barcodes.barcodeset.xml prefix.consensusreadset.xml

If you do not need to import the demultiplexed data into SMRT Link, it is advised
to use `--no-pbi`, omit pbi index file, to minimize memory consumption and time to result.

### *Symmetric* or *Tailed* options

    Raw: --same
    CCS: --same --ccs

### *Asymmetric* options

    Raw: --min-passes 1
    CCS: --ccs

## Input data
Input data is either raw unaligned subreads, straight from a Sequel, or
unaligned ccs reads, generated by [CCS 2](https://github.com/PacificBiosciences/unanimity);
both in the PacBio enhanced BAM format. If you want to demux RSII data, please
use SMRT Link or bax2bam to convert h5 to BAM.

Barcodes are provided as a FASTA file, one entry per barcode sequence,
**no duplicate** sequences,
orientation agnostic (forward or reverse-complement, but **NOT** reversed).
Example:

    >bc1000
    CTCTACTTACTTACTG
    >bc1001
    GTCGTATCATCATGTA
    >bc1002
    AATATACCTATCATTA

Please name your barcodes with an alphabetic character prefix to avoid
later confusion of barcode name and index.

## Workflow
*Lima* processes input reads grouped by ZMW, except if `--per-read` is chosen.
All barcode regions along the read are processed individually.
The final per-ZMW result is a [summary](#implementation-details) over all barcode regions,
a pair of selected barcodes from the provided set of candidate barcodes;
subreads from the same ZMW will have the same barcode and barcode quality.
For a particular target barcode region, every barcode sequence gets aligned
as given and as reverse-complement, and higher scoring orientation is chosen;
result is a list of scores over all candidate barcodes.

If only *same* barcode pairs are of interest, *symmetric/tailed*, please use
`--same` to filter out *different* barcode pairs.

If only *different* barcode pairs are of interest, *asymmetric*, please use
`--min-passes 1` to require at least two barcodes to be read.

## Output
*Lima* generates multiple output files per default, all starting with the same
prefix as the output file, omitting suffixes `.bam`, `.subreadset.xml`, and
`.consensusreadset.xml`. The report infix is `lima`.
Example:

    lima m54007_170702_064558.subreads.bam barcode.fasta /my/path/m54007_170702_064558_demux.subreadset.xml

For all output files, the prefix will be `/my/path/m54007_170702_064558_demux.`

### BAM
The first file `prefix.bam` contains clipped records, annotated with
barcode tags, that passed filters and respect option `--same`.

### Report
Second file is `prefix.lima.report`, a tab-separated file about each ZMW, unfiltered.
This report contains any information necessary to investigate the demultiplexing
process and the underlying data.
A single row contains all reads of a single ZMW. For `--per-read`, each row
contains one subread and ZMWs might span multiple rows.

### Summary
Third file is `prefix.lima.summary`, shows how many ZMWs have been filtered,
how ZMWs many are *same/different*, and how many reads have been filtered.

    ZMWs input                (A) : 6937
    ZMWs above all thresholds (B) : 1565
    ZMWs below any threshold  (C) : 5372

    Marginals for (C):
    ZMWs below min length         : 1 (0%)
    ZMWs below min score          : 0 (0%)
    ZMWs below min passes         : 0 (0%)
    ZMWs below min score lead     : 59 (1%)
    ZMWs w/o adapter              : 5312 (99%)

    For (B):
    ZMWs w/ same barcode          : 1552 (99%)
    ZMWs w/ different barcodes    : 13 (1%)

    For (A):
    ZMWs allow diff barcode pair  : 709 (10%)
    ZMWs allow same barcode pair  : 1625 (23%)

    For (B):
    Reads above length            : 4567 (100%)
    Reads below length            : 5 (0%)

Explanation of each block:

1) Number of ZMWs that went into lima,
   how many ZMWs have been passed into the output file,
   and how many did not qualify.

2) For those ZMWs that did not qualify,
   the marginal counts of each filter; each filter is explained in great detail
   elsewhere in this document.

3) For those ZMWs that passed, how many have been flagged as having a
   *same* or *different* barcode pair.

4) For all input ZMWs, how many allow calling a *same* or *different*
   barcode pair. This a simplified version of, how many ZMW have at least one
   full pass to allow a *different* barcode pair call and how many ZMWs have
   at least half an adapter, allowing a *same* barcode pair call.

5) For those ZMWs that qualified, list the number of reads that are above and
   below the provided `--min-length` threshold  (details see [here](#filter)).

### Counts
Fourth file is `prefix.lima.counts`, a tsv file, that shows the counts of each
observed barcode pair; only passing ZMWs are counted.
Example:

    $ column -t prefix.lima.counts
    IdxFirst  IdxCombined      IdxFirstNamed  IdxCombinedNamed  Counts
    1         1                bc1002         bc1002            113
    14        14               bc1015         bc1015            129
    18        18               bc1019         bc1019            106

### Clipped regions
Using the option `--dump-clips`, clipped barcode regions are stored in the file
`prefix.lima.clips`.
Example:

    $ head -n 6 prefix.lima.clips
    >m54007_170702_064558/4850602/6488_6512 bq:34 bc:11
    CATGTCCCCTCAGTTAAGTTACAA
    >m54007_170702_064558/4850602/6582_6605 bq:37 bc:11
    TTTTGACTAACTGATACCAATAG
    >m54007_170702_064558/4916040/4801_4816 bq:93 bc:10

### Removed records
Using the option `--dump-removed`, records that did not pass provided thresholds
or are without barcodes, are stored in the file `prefix.lima.removed.bam`.
*Lima* does not generate a `.pbi`, nor dataset for this file.

This option cannot be used with any splitting option.

### DataSet
One DataSet, SubreadSet or ConsensusReadset, is generated per output BAM file.

### PBI
One PBI file is generated per output BAM file.

## Filter
*Lima* offers a set of options for processing, including trivial and
 sophisticated filters.

### `--min-length`
Reads with length below `N` bp after demultiplexing are omitted. The default is `50`.
ZMWs with no reads passing are omitted.

### `--max-input-length`
Reads with length above `N` bp are omitted for scoring in the demultiplexing
step. The default is `0`, meaning deactivated.

### `--min-score`
ZMWs with barcode score below `N` are omitted. The default is `0`.
It is advised to set it to `45`.

### `--min-passes`
ZMWs with less than `N` full passes, a read with a leading and
trailing adapter, are omitted. Default is `0`, no full-pass needed. Example

    0 pass  : insert - adapter - insert
    1 pass  : insert - adapter - INSERT - adapter - insert
    2 passes: insert - adapter - INSERT - adapter - INSERT - adapter - insert

### `--max-scored-barcodes`
Only use up to `N` barcode pair regions to find the barcode.
Default is `0`, deactivated filter. This is equivalent to the
maximum number of scored adapters in bam2bam.

### `--score-full-pass`
Only use reads flanked by adapters on both sides for barcode identification,
full-pass reads.

### `--keep-idx-order`
Per default, the two identified barcode idx are sorted ascending, as in raw data,
the correct order cannot be determined. This affects the `bc` tag, `prefix.counts`
file, and `--split-bam` file names; `prefix.report` columns `IdxLowest`,
`IdxHighest`, `IdxLowestNamed`, `IdxHighestNamed` will have same order as
`IdxFirst` and `IdxCombined`.

If you are using an asymmetric barcode design with `NxN` pairs
and your input is CCS, you can use `--keep-idx-order` to preserve
the order. If your input is raw subreads and you use `NxN` asymmetric pairs,
there is no way to distinguish between pairs `bc101--bc102` and `bc102--bc101`.

### `--per-read`
Score and tag per subread, instead per ZMW.

### `--window-size-mult`
The candidate region size multiplier: `barcode_length * multiplier`, default `1.5`.
Optionally, you can specify the region size in base pairs with `--window-size-bp`.

### Alignment options

    -A,--match-score        Score for a sequence match.
    -B,--mismatch-penalty   Penalty for a mismatch.
    -D,--deletion-penalty   Deletions penalty.
    -I,--insertion-penalty  Insertion penalty.
    -X,--branch-penalty     Branch penalty.

### `--ccs`
Set defaults to `-A 1 -B 4 -D 3 -I 3 -X 1`

### `--peek`
The option `--peek N` allows to look at the first `N` ZMWs of the input and
return the mean barcode score. This allows to test multiple test `barcode.fasta`
files and see which set of barcodes has been used.

### `--guess`
The option `--guess N` performs demultiplexing twice. In the first iteration,
all barcodes are tested per ZMW. Afterwards, the barcode occurrences are counted
and their mean barcode score is tested against the provided threshold `N`;
only those barcode pairs that pass this threshold are used in the second
iteration to produce the final demux output. A `prefix.lima.guess` file shows
the decision process; `--same` is being respected.
Example:

    $ column -t *guess
    IdxFirst  IdxCombined  IdxFirstNamed  IdxCombinedNamed  NumZMWs  MeanScore  Picked
    0         0            bc1002         bc1002            174      76         1
    0         4            bc1002         bc1048            1        43         0
    9         9            bc1080         bc1080            3        16         0
    10        10           bc1093         bc1093            742      75         1
    10        14           bc1093         bc1115            2        55         1
    12        12           bc1101         bc1101            4        18         0

### `--guess-min-count`
The minimum ZMW abundance to whitelist a barcode. This filter is `AND` with
the minimum barcode score provided by `--guess`. Default 0.

### `--peek` && `--guess`
The optimal way is to use both advanced options in combination, e.g.,
`--peek 1000 --guess 45`. Lima will run twice on the input data.
For the first 1000 ZMWs, *lima* will guess the barcodes and store the mask of
identified barcodes.
In the second run, the barcode mask is used to demultiplex all ZMWs.

### `--single-side`
Identify barcodes in molecules that only have barcodes adjacent to one adapter.

### `--enforce-first-barcode`
Set the first barcode to be barcode index 0.

### `--var-barcode-length`
Allow barcode to have different lengths. This will increase run time.

## Mechanical options
### `--num-threads`
Spawn a threadpool of `N` threads, `0` means all available cores. Default is `0`.
This option also controls the number of threads used for BAM and PBI compression.

### `--chunk-size`
By default, each thread consumes `N` ZMWs per chunk for processing. Default is `10`.

### `--no-bam`
Do not produce BAM output, nor PBI. Useful, if only reports are of interest,
as time to results is lower.

### `--no-reports`
Do not produce any reports. Interesting, if only demultiplexed BAM is of interest.

## Quality Control
For quality control, we offer three R scripts to help you troubleshoot your
data. The first two are for low multiplex data.
The third is for high plex data, easily showing 384 barcodes.

### `report_detail.R`

The first is for the `lima.report` file: `scripts/r/report_detail.R`. Example:

    $ Rscript --vanilla scripts/r/report_detail.R prefix.lima.report

Optionally the second argument is the output file type `png` or `pdf`, with
`png` as default:

    $ Rscript --vanilla scripts/r/report_detail.R prefix.lima.report pdf

You can also restricts output to only barcodes of interest, using the barcode
name not the index.
For example, all barcode pairs that contain the barcode "bc1002":

    $ Rscript --vanilla scripts/r/report_detail.R prefix.lima.report png bc1002

A specific barcode pair "bc1020--bc1045"; remark, the script will look for both
combinations "bc1020--bc1045" and "bc1045--bc1020":

    $ Rscript --vanilla scripts/r/report_detail.R prefix.lima.report png bc1020--bc1045

Or any combination of those two:

    $ Rscript --vanilla scripts/r/report_detail.R prefix.lima.report pdf bc1002 bc1020--bc1045 bc1321

#### Yield
Per barcode base yield:
<img src="img/plots/detail_yield_base.png" width="1000px">

Per barcode read yield:
<img src="img/plots/detail_yield_read.png" width="1000px">

Per barcode ZMW yield:
<img src="img/plots/detail_yield_zmw.png" width="1000px">

#### Scores
Score per number of adapters (lines) and all adapters (histogram).
[What are half adapters?](#what-are-half-adapters)
<img src="img/plots/detail_scores_per_adapter.png" width="1000px">

Read length vs mean score (99.9% percentile)
<img src="img/plots/detail_read_length_vs_score.png" width="1000px">

HQ length vs mean score (99.9% percentile)
<img src="img/plots/detail_hq_length_vs_score.png" width="1000px">

#### Read length (99.9% percentile, 1000 binwidth)
Grouped by barcode, same y axis :
<img src="img/plots/detail_read_length_hist_group_same_y.png" width="1000px">

Grouped by barcode, free y axis:
<img src="img/plots/detail_read_length_hist_group_free_y.png" width="1000px">

Not grouped into facets, line histogram:
<img src="img/plots/detail_read_length_linehist_nogroup.png" width="1000px">

Barcoded vs non-barcoded:
<img src="img/plots/detail_read_length_hist_barcoded_or_not.png" width="1000px">

#### HQ length (99.9% percentile, 2000 binwidth)
Grouped by barcode, same y axis:
<img src="img/plots/detail_hq_length_hist_group_same_y.png" width="1000px">

Grouped by barcode, free y axis:
<img src="img/plots/detail_hq_length_hist_group_free_y.png" width="1000px">

Not grouped into facets, line histogram:
<img src="img/plots/detail_hq_length_linehist_nogroup.png" width="1000px">

Barcoded vs non-barcoded:
<img src="img/plots/detail_hq_length_hist_barcoded_or_not.png" width="1000px">

#### Adapters  (99.9% percentile, 1 binwidth)
Number of adapters:
<img src="img/plots/detail_num_adapters.png" width="1000px">

### `counts_detail.R`

The second is for multiple `counts` files: `scripts/r/counts_detail.R`.
This script cannot be called from the CLI yet, to be implemented.

#### ZMW Counts
Single plot:
<img src="img/plots/counts_nogroup.png" width="1000px">

Grouped by barcode into facets:
<img src="img/plots/counts_group.png" width="1000px">

### `report_summary.R`
Third script is for high-plex data in one `lima.report` file:
`scripts/r/report_summary.R`.

Example:

    $ Rscript --vanilla scripts/r/report_summary.R prefix.lima.report

Yield per barcode:
<img src="img/plots/summary_yield_zmw.png" width="750px">

Score distribution across all barcodes:
<img src="img/plots/summary_score_hist.png" width="750px">

Score distribution per barcode:
<img src="img/plots/summary_score_hist_2d.png" width="1000px">

Read length distribution per barcode:
<img src="img/plots/summary_read_length_hist_2d.png" width="1000px">

HQ length distribution per barcode:
<img src="img/plots/summary_hq_length_hist_2d.png" width="1000px">


## FAQ
### Why *lima* and not bam2bam?
*Lima* was developed to provide a better user experience. Both use the
identical core alignment step, but the algorithm to identify a barcode pairs
and usability has been improved.

### What is the barcode score
The barcode score is an indicator how well the chosen barcode pair matches.
It is normalized to a range between 0 and 100, whereas 0 is no hit
and 100 perfect match.

### How fast is fast?
Example: 200 barcodes, asymmetric mode (try each barcode forward and
reverse-complement), 300,000 CCS reads. On my 2014 iMac with 4 cores + HT:

    503.57s user 11.74s system 725% cpu 1:11.01 total

Those 1:11 minutes translate into 0.233 milliseconds per ZMW,
1.16 microseconds per barcode for both sides aligning forward and reverse-complement,
and 291 nanoseconds per alignment. This includes IO.

### *Lima* does not utilize the maximal number of provided cores
This might be a simple IO bottleneck. With a barcode.fasta containing only a few
barcodes, most of the time is spent reading and writing BAM files, as the barcode
identification is too fast.

### Why is the memory consumption really high?
The on-the-fly `.bam.pbi` file generation needs to buffer the output data. If you
do not need a `.bam.pbi` file for SMRT Link import, use `--no-pbi` to decrease
memory usage to a minimum and accelerate time to result.

### Is there a way to show the progress?
No. Please run `wc -l prefix.report` to get the number of already processed ZMWs.

### Can I split my data by barcode?
You can either iterate over the `prefix.bam` file N times or use
`--split-bam`. Each barcode has its own BAM file called
`prefix.idxBest-idxCombined.bam`, e.g., `prefix.0-0.bam`.

Optionally `--split-bam-named`, names the files by their barcode names instead
of their barcode indices.

This mode consumes more memory, as output cannot be streamed.

### What are half adapters?
If there is an adapter call with only one barcode region,
as the high-quality region finder cut right through the adapter,
the preceeding or succeeding subread was too short and got removed,
or the sequencing reaction started/stopped there, we call such an adapter half.
Thus, there are also 1.5, 2.5, N+0.5 adapter calls.

ZMWs with half or only one adapter can be used to identify *same* barcode pairs;
positive-predictive value might be reduced compared to high adapter calls.
For asymmetric designs with *different* barcodes in a pair, at least a single
full-pass read is required; this can be two adapters, two half-adapters, or a
combination.

### Why are *different* barcode pair hits reported in --same mode?
*Lima* tries all barcode combinations and `--same` only filters BAM output.
Sequences flanked by *different* barcodes are still reported, but are not
written to BAM. By not enforcing only *same* barcode pairs, *lima* gains
higher PPV, as your sample might be contaminated and contains unwanted
barcode pairs; instead of enforcing one *same* pair, *lima* rather
filters such sequences. Every *symmetric* / *tailed* library contains few *asymmetric*
templates. If many *different* templates are called, your library preparation
might be bad.

### Why are *same* barcode pair hits reported in the default *different* mode?
Even if your sample is labeled *asymmetric*, *same* hits are simply
sequences flanked by the *same* barcode ID.

But my design does not include *same* barcode pairs! We are aware of this,
but it happens that some ZMWs do not have sufficient signal to call a pair
with different barcodes.

### How do barcode indices correspond to the input sequences?
Input barcode sequences are tagged with an incrementing counter. The first
sequence is barcode `0` and the last barcode `numBarcodes - 1`.

### I used the tailed library prep, what options to choose?
Use `--same`.

### How can I demultiplex data with one adapter only being barcoded?
Use `--single-side`.

### What is different to bam2bam?
 * CCS read support
 * Barcodes of every adapter gets scored for raw subreads
 * Do not enforce symmetric barcode pairing, which increases PPV
 * For asymmetric barcodes, `lima` can report the identified order, instead of
   ascending sorting
 * Call barcodes per barcode region and do not enforce adapter coupling
 * Open-source and can be compiled on your local Mac or Linux machine
 * Nice reports for QC