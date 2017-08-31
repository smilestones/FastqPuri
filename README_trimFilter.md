# trimFilter user manual

 This program reads a `fastq` file as an input, and filters it according 
 to the following criteria:
 - Discard/trims reads containing adapter remnants.
 - Discards reads matching contaminations (sequences collected in a `fasta`
   file or in an `idx` file created by `makeTree`, `makeSA`, `makeBloom` 
 - Discards/trims low quality reads.
 - Discards/trims reads containing N base callings.

## Running the program


```
Usage: trimFilter --ifq <INPUT_FILE.fq> --length <READ_LENGTH>
                  --output [O_PREFIX]
                  --adapters [<ADAPTERS.fa>:<mismatches>:<score>]
                  --method [TREE|SA|BLOOM]
                  (--idx [<INDEX_FILE>:<score>:<lmer_len>] |
                   --ifa [<INPUT.fa>:<score>:[lmer_len]])
                  --trimQ [NO|ALL|ENDS|FRAC|ENDSFRAC|GLOBAL]
                  --minL [MINL]  --minQ [MINQ]
                  (--percent [percent] | --global [n1:n2])
                  --trimN [NO|ALL|ENDS|STRIP]
Reads in a fq file (gz, bz2, z formats also accepted) and removes:
  * low quality reads,
  * reads containing N base callings,
  * reads representing contaminations, belonging to sequences in INPUT.fa
Outputs 4 [O_PREFIX]_fq.gz files containing: "good" reads, discarded
low Q reads discarded reads containing N's, discarded contaminations.
Options:
 -v, --version prints package version.
 -h, --help    prints help dialog.
 -f, --ifq     fastq input file [*fq|*fq.gz|*fq.bz2], mandatory option.
 -l, --length  read length: length of the reads, mandatory option.
 -o, --output  output prefix (with path), optional (default ./out).
 -A, --adapter adapter input three fields separated by colons:
               <ADAPTERS.fa>: fasta file containing adapters,
               <mismatches>: maximum mismatch count allowed,
               <score>: score threshold  for the aligner.
 -x, --idx     index input file. To be included with any method. 3 fields
               3 fields separated by colons: 
               <INDEX_FILE>: output of makeTree makeSA, makeBloom,
               <score>: score threshold to accept a match [0,1],
               [lmer_len]: correspond to the length of the lmers to be 
                        looked for in the reads [1,READ_LENGTH].
 -a, --ifa     fasta input file. To be included only with method TREE
               (it excludes the option --idx). Otherwise, an
               index file has to be precomputed and given as parameter
               (see option --idx). 3 fields separated by colons: 
               <INPUT.fa>: fasta input file [*fa|*fa.gz|*fa.bz2],
               <score>: score threshold to accept a match [0,1],
               <lmer_len>: depth of the tree: [1,READ_LENGTH]. It will 
                        correspond to the length of the lmers to be 
                        looked for in the reads.
 -C, --method  method used to look for contaminations: 
               TREE:  uses a 4-ary tree. Index file optional,
               SA:    uses a suffix array. Index file mandatory,
               BLOOM: uses a bloom filter. Index file mandatory.
 -Q, --trimQ   NO:       does nothing to low quality reads (default),
               ALL:      removes all reads containing at least one low
                         quality nucleotide.
               ENDS:     trims the ends of the read if their quality is
                         below the threshold -q,
               FRAC:     discards a read if the fraction of bases whose
                         quality lies below 
                         the threshold -q is over 5 percent or a user 
                         defined percentage in -p.
               ENDSFRAC: trims the ends and then discards the read if 
                         there are more low quality nucleotides than the
                         allowed by the option -p.
               GLOBAL:   removes n1 cycles on the left and n2 on the 
                         right, specified in -g.
               All reads are discarded if they are shorter than MINL.
 -m, --minL    minimum length allowed for a read before it is discarded
               (default 25).
 -q, --minQ    minimum quality allowed (int), optional (default 27).
 -p, --percent percentage of low quality bases to be admitted before 
               discarding a read (default 5), 
 -g, --global  required option if --trimQ GLOBAL is passed. Two int,
               n1:n2, have to be passed specifying the number of cycles 
               to be globally cut from the left and right, respectively.
 -N, --trimN   NO:     does nothing to reads containing N's,
               ALL:    removes all reads containing N's,
               ENDS:   trims ends of reads with N's,
               STRIPS: looks for the largest substring with no N's.
               All reads are discarded if they are shorter than minL.
```


## Output description

- `O_PREFIX_good.fq.gz`: contains reads that passed all filters (maybe trimmed).
- `O_PREFIX_adap.fq.gz`: contains discarded due to the presence of adapters.
- `O_PREFIX_cont.fq.gz`: contains contamination reads.
- `O_PREFIX_lowQ.fq.gz`: contains reads discarded due to low quality issues.
- `O_PREFIX_NNNN.fq.gz`: contains reads discarded due to *N*'s issues.
- `O_PREFIX_summary.bin`: binary file where information about the filtering 
   process is stored. Structure of the file. 
    * tilters, `4*sizeof(short)  Bytes`: array of shorts with entries 
       i = {ADAP(0), CONT(1), LOWQ(2), NNNN(3)}. A given entry takes 
       the value 1 if the filter was applied and 0 otherwise.
    * trimmed, `4*sizeof(int) Bytes`: array of integers with entries 
       i = {ADAP(0), CONT(1), LOWQ(2), NNNN(3)}, containing how many 
       reads were trimmed due to the corresponding filter.  
    * discarded, `4*sizeof(int) Bytes`: array of integers with entries 
       i = {ADAP(0), CONT(1), LOWQ(2), NNNN(3)}, containing how many 
      reads were discarded due to the corresponding filter. 
    * good, `sizeof(int) Bytes`: number of accepted reads (maybe trimmed).
    * nreads, `sizeof(int) Bytes`: total number of reads. 

## Filters

#### Adapters 

TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO 

#### Impurities  

 Contaminations are removed if a fasta file is given as an input. 
 Three methods have been implemented to check for contaminations: 

- **TREE**: this method is designed to identify impurities from 
  small sequences, such as rRNA, or EColi (not larger than 10MB).
  When using this method, ont has to pass one of these two options: 
   - `--ifa <INPUT.fa>:<score>:<lmer_len>`: the file `INPUT.fa`
     is read and a tree of depth `lmer_len` is constructed on the flight. 
   - `--idx <INDEX_FILE>:<score>`: in `INDEX_FILE`  the tree structure 
      was stored with `./makeTree`.
  A read will be considered to be a contamination if the score is bigger 
  than `score`. The score is computed as the proportion of Lmers of a given
  read found in the tree. This method is very fast, every search is 
  `O( 2 * Lmer * (L - Lmer + 1))` (the reverse complement has to be
   checked also), but has the drawback that is very memory intensive. 
   This is why we have limited the construction of a tree to sequences 
   whose size does not exceed 10 MB. Constructing a tree is very fast, 
   that is why most of the times it might be not worth it to create the 
   index file and then read it for every sample. Nevertheless, both 
   options are provided.
- **SA**: this method is designed for sequences of medium size (several 
  hundreds of MB). An index file has to be precomputed in a previous 
  step, using `makeSA`, so that we do not have to compute it for 
  every sample. The following option has to be passed: 
   - `--idx <INDEX_FILE>:<score>:<lmer_len>`
  indicating the index file name, the score threshold and the 
  length of the Lmers. A suffix array is constructed using an analogous 
  strategy to that of `STAR` (see more details in `README_makeSA.md`), 
  and then a binary search for exact matches is performed with the help
  of a predix index of `m` bases, so that we do not have to jump so much 
  within a binary search. The complexity of a search is: `O(log(Lgenome)*2*L)`,
  where L is the read length. **WORK IN PROGRESS**
- **BLOOM**: this method is designed for large sequences up to 4GB. An 
  index is precomputed, using `makeBloom`, so that it is computed only 
  once for all samples. The index file name and the threshold score have 
  to be specified through the following option: 
   - `--idx <INDEX_FILE>:<score>`
  The score is computed as for the previous options. **WORK IN PROGRESS**

#### LowQ

- `--trimQ NO` or flag absent: nothing is done to the reads with low quality.
- `--trimQ ALL`: all reads containing at least one low quality nucleotide are
  redirected to  `*_lowq.fq.gz`
- `--trimQ ENDS`: look for low quality (below MINQ) base callings at the 
  beginning and at the end of the read. Trim them at both ends until the 
  quality is above the threshold. Keep the read in `*_good.fq.gz`
  and annotate in the fourth line where the read has been trimmed (starting to
  count from 0) if the length of the remaining part is larger than `MINL`.
  Redirect the read to `*_lowq.fq.gz` otherwise.
    Examples (-q 27 [<] -m 25): 
 ```
 ORIGINAL                                            FILTERED
 @ read 1081133                                      @ read 1081133
 GTACATTGATGAGCAACATGGGGCTTGAACTGGCGCTGAAACAGTTAGGA  GTACATTGATGAGCAACATGGGGCTTGAACTGGCGCTGAAACAGTTAGG
 +position: -144740                                  +position: -144740 TRIMQ:0:48
 IIIIIIIIIIIII9IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII9  IIIIIIIIIIIII9IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII
 @ read 3435                                         @ read  3435 
 CAGTTCTTTGGTGTAGATGGAGCAGGACGGGATACCCATACCGTGACCCA  CTTTGGTGTAGATGGAGCAGGACGGGATACCCATACCGTG
 +position: -19377                                   +position: -19377  TRIMQ:5:44
 99999IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII99999  IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII
 @ read 108110                                       @ read 108110 
 CAGATAAATGCGAATGAATGGGTAACACTGGAGCTTTTAATGGCTGTAAC  AGATAAATGCGAATGAATGGGTAACACTGGAGCTTTTAATGGCTGTAA
 +position: -4142524                                 +position: -4142524  TRIMQ:1:48
 9IIIIIIIIIIIIIIIIIII9IIII9III9IIIIIIIIIIIIIIIIIII9  IIIIIIIIIIIIIIIIIII9IIII9III9IIIIIIIIIIIIIIIIIII
 @ read 108111                                      
 AGCGTGACGGTGTAACGCCCGCCTTTTGATGACTGGGTTTCAAAGAAACG  
 +position: -3336785                                 
 999999999999999IIIIIIII9IIIIIIIII9II99999999999999  
 ```

- `--trim FRAC [--percent p]`: redirect  the read to `*_lowq.fq.gz` if 
   there are more than `p%` nucleotides whose quality lies below the threshold. 
   `p=5` per default.  
    Examples (-q 27 [<] -m 25 -p 5): 
 ```
 ORIGINAL                                            FILTERED
 @ read 1081133                                      @ read 1081133
 GTACATTGATGAGCAACATGGGGCTTGAACTGGCGCTGAAACAGTTAGGA  GTACATTGATGAGCAACATGGGGCTTGAACTGGCGCTGAAACAGTTAGGA
 +position: -144740                                  +position: -144740 
 IIIIIIIIIIIII9IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII9  IIIIIIIIIIIII9IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII9
 @ read 3435                                         
 CAGTTCTTTGGTGTAGATGGAGCAGGACGGGATACCCATACCGTGACCCA  
 +position: -19377                                   
 99999IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII99999  
 @ read 108110                                       
 CAGATAAATGCGAATGAATGGGTAACACTGGAGCTTTTAATGGCTGTAAC  
 +position: -4142524                                 
 9IIIIIIIIIIIIIIIIIII9IIII9III9IIIIIIIIIIIIIIIIIII9  
 @ read 108111                                      
 AGCGTGACGGTGTAACGCCCGCCTTTTGATGACTGGGTTTCAAAGAAACG  
 +position: -3336785                                 
 999999999999999IIIIIIII9IIIIIIIII9II99999999999999  
 ```
- `--trim ENDSFRAC --percent p`: first trim the ends as in the `ENDS` option. 
   Accept the trimmed read if the number of low quality nucleotides does not 
   exceed `p%` (default  `p = 5`).
   Redirect the read to `*_lowq.fq.gz` otherwise.
   Examples (-q 27 [<] -m 25  -p 5):
   Missing reads would land in `*_lowq.fq.gz`.  
 ```
 ORIGINAL                                            FILTERED
 @ read 1081133                                      @ read 1081133
 GTACATTGATGAGCAACATGGGGCTTGAACTGGCGCTGAAACAGTTAGGA  GTACATTGATGAGCAACATGGGGCTTGAACTGGCGCTGAAACAGTTAGG
 +position: -144740                                  +position: -144740 TRIMQ:0:48
 IIIIIIIIIIIII9IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII9  IIIIIIIIIIIII9IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII
 @ read 3435                                         @ read  3435 
 CAGTTCTTTGGTGTAGATGGAGCAGGACGGGATACCCATACCGTGACCCA  CTTTGGTGTAGATGGAGCAGGACGGGATACCCATACCGTG
 +position: -19377                                   +position: -19377  TRIMQ:5:44
 99999IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII99999  IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII
 @ read 108110                                       
 CAGATAAATGCGAATGAATGGGTAACACTGGAGCTTTTAATGGCTGTAAC  
 +position: -4142524                                 
 9IIIIIIIIIIIIIIIIIII9IIII9III9IIIIIIIIIIIIIIIIIII9  
 @ read 108111                                      
 AGCGTGACGGTGTAACGCCCGCCTTTTGATGACTGGGTTTCAAAGAAACG  
 +position: -3336785                                 
 999999999999999IIIIIIII9IIIIIIIII9II99999999999999  
 ```

- `--trim GLOBAL --global n1:n2`: cut all read globally `n1` nucleotides from 
   the left and `n2` from the right. 

**Note:** qualities are evaluated assuming the reads to follow the
L - Illumina 1.8+ Phred+33, convention, see [Wikipedia](https://en.wikipedia.org/wiki/FASTQ_format#Encoding).
Adjust the values for a different convention.            


#### N trimming 

We allow for the following options: 

- `--trimN NO` (or flag absent): Nothing is done to the reads containing N's. 
- `--trimN ALL`: All reads containing at least one N are redirected to 
  `*_NNNN.fq.gz` 
- `--trimN ENDS`: N's are trimmed if found at the ends, left "as is" 
  otherwise. If the trimmed read length is smaller than MINL, it is discarded. 
  Example: 
```
ORIGINAL                                           FILTERED
@ read 1037                                        @ read 1037
NNTCGACACAAGAAAATGCGCCAATTTTGAGCCAGACCCCAGTTACGCNN TCGACACAAGAAAATGCGCCAATTTTGAGCCAGACCCCAGTTACGC
+position: -2044610                                +position: -2044610  TRIMN:2:47
IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII
@ read 1038                                        
AAAAAACGGTCGTGGTNGGCTACTGTTATTAAAGCGTTGGCTACAAAAAG 
+position: 1361068                                 
IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII 
@ read 1039                                        
CCAACGGACNATGGAAGTGCTNCGGCTGGNGTTTTTATCCTCCGGCATTC 
+position: -4282223                                
IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII 
@ read 1043                                        @ read 1043 
AATCTGAAACGAAANCTGACAGCGNNCCCCGCTTCTGACAAAATAGGCGN AATCTGAAACGAAANCTGACAGCGNNCCCCGCTTCTGACAAAATAGGCG
+position: -4685876                                +position: -4685876  TRIMN:0:48
IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII
```


- `--trimN STRIP`: Obtain the largest N free subsequence of the read. Accept it 
   if is longer than the half of the original read length, redirect it to 
   `*_NNNN.fq.gz` otherwise. Example:
```
ORIGINAL                                           FILTERED
@ read 1037                                        @ read 1037
NNTCGACACAAGAAAATGCGCCAATTTTGAGCCAGACCCCAGTTACGCNN NNTCGACACAAGAAAATGCGCCAATTTTGAGCCAGACCCCAGTTACGCNN
+position: -2044610                                +position: -2044610  TRIMN:2:47 
IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII
@ read 1038                                        @ read 1038 
AAAAAACGGTCGTGGTNGGCTACTGTTATTAAAGCGTTGGCTACAAAAAG GGCTACTGTTATTAAAGCGTTGGCTACAAAAAG
+position: 1361068                                 +position: 1361068  TRIMN:17:49
IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII
@ read 1039 
CCAACGGACNATGGAAGTGCTNCGGCTGGNGTTTTTATCCTCCGGCATTC 
+position: -4282223                                
IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII 
@ read 1043                                         
AATCTGAAACGAAANCTGACAGCGNNCCCCGCTTCTGACAAAATAGGCGN
+position: -4685876 
IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII
```


## Test/examples
 The example in folder `examples/trimFilter_FReport/` works in the following 
 way (**NOTE**: this has to be MUCH more detailed):                        
 The code following way (**NOTE**: this has to be MUCH more detailed):   
                                                                         
 1. `EColi_rRNA.fq` was created with `./create_fq.sh`. The file contains:
    * 2e5 reads of length 50 from `EColi_genome.fa` with NO errors.      
    * 5e4 reads of length 50 from `rRNA_modified.fa` with NO errrors     
      (rRNA contaminations).                                             
    * Artificially generated reads with low quality score (see script).  
    * Artificially generated reads with Ns (see script).                 
 2. The code was tested with flags:                                      
    `../../bin/trimFilter -l 50 --ifq EColi_rRNA.fq.gz --method TREE\    
     --ifa rRNA_CRUnit.fa:0.9:50 --trimQ ENDSFRAC --trimN STRIP`         
    i.e., we check for contaminations from rRNA, trim reads with lowQ at 
    the ends and less than 5% in the remaining part, and strip reads     
    containing N's. Output is stored in `example*.fq.gz` files.          
 3. One can run the example in `./run_example.sh` that will generate the 
    output again with the prefix `tmp` (**WARNING:** do not run          
    `./create_fq.sh` again if you want the same results).                
 4. With this set up, it is possible to run further customized tests.    
                                                                         
## Contributors

Paula Pérez Rubio 

## License

GPL v3 (see LICENSE.txt)