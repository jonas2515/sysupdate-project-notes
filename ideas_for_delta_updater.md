## Little glossary:

- we have 3 images:
  - currently booted image: image of the slot we are currently booted into; can't be overwritten, but can be read from
  - stale image: image that we're not currently booted into, the one we must override
  - fresh image: image we're going to download and write

- most recent installed image: either currently booted image, or the stale image, chances are high that it's the former 

## Challenges and Requirements

#### slow things involved in the process:

  - downloading: most likely the biggest bottleneck, esp if we only download a 4KiB(?) block at a time, and using TCP (likely won't saturate the avail bandwidth). 
    - multiple (non-adjacent) ranges can be requested at the same time, likely makes sense to batch multiple blocks into a bigger download to get TCP to sature the link
    - also possible: multiple parallel small downloads at the same time, but probably the former is better
  - reading from disk: likely quite fast, parallelizing doesn't make it faster
  - hashing: likely quite fast, makes sense to parallelize, but with an upper limit of maybe 30%(?) of total CPU resources 
  - writing to disk: likely quite fast, parallelizing doesn't make it faster

#### requirements / nice to have:

  - check of whether update is available must be fast: No problem, we can compare versions
  - knowing the download size beforehand without using tons of CPU would be nice
    - comparing two manifest files instead of actually hashing the blocks ourselves would be super fast
      - still need to read all the blocks when we do the actual update in order to catch disk corruption (even with DM-Verity, we can only catch corruption when actually reading a block)
  - killing the updater half-way may never break the currently booted image
  - would be nice to make the updater "naturally" able to pick up if it was stopped/killed in the middle
  - cater for moving identical blocks to different positions between images


## Approaches:

### 1) Super simple approach

#### steps:
- download manifest file for "fresh image" with hashes for each block
- while-loop going over each block of manifest file for "fresh image"
  - do a hash of block in "most recent installed image"
  - compare hash to "fresh image" block hash
  - if hashes match
    - if "most recent installed image" == "currently booted image" -> copy block over into the "stale image"
    - if "most recent installed image" == "stale image" -> just continue to next block
  - if hashes don't match, download the block into tmp dir using HTTP range request and write block to "stale image"
  - need to do a read-back to check the hash, this should also force the actual write

- after the while loop ran completely, we mark the "stale image" as next image to boot

#### downsides:
- performance for updating is slow
  - likely downloading will be a big bottleneck
    - we're downloading only one block at a time (so 4KiB?)
  - no downloading + reading hashing + writing at the same
- performance for checking download size is slow
  - needs to do all the hashes

#### upsides:
- super simple to implement


### 2) Simple but with quick update checks

#### step a) check download size:
- download manifest file for "most recent installed image"
- download manifest file for "fresh image"
- while-loop going over each block of manifest file for "fresh image"
  - compare hash to downloaded manifest file for "most recent installed image" (this is super quick)

#### step b) do the update:
- basically same as in 1)

#### possible optimizations:
- download all the new blocks from step a) in a single go and write the fresh blocks into the "stale image" while downloading
  - this should increase download speeds by a lot
  - afterwards still need to read-back and hash+compare all the blocks to confirm that the update worked
    - if hashes don't match, collect them, and jump to download + write step again
    - should also be fairly easy to parallelize the hashing here
- using the hash format of DM-Verity itself for manifests
  - then we could compare to those hashes rather than comparing to the manifest for "most recent installed image"
    - has the upside of being able to naturally pick up again when the updater was interrupted before, without re-downloading stuff
  - possibly wouldn't even have to read-back anymore to confirm that everything was written, and could rather compare new DM verity hashes after updating

#### downsides:
- download size can end up larger than what we tell the user if there are corrupted blocks

#### upsides:
- checking download size is super fast


### 3) A bit more complicated, catering for moved around blocks

#### step a) check download size:
- download manifest file for "currently booted image"
- download manifest file for "fresh image"
- while-loop going over each block of manifest file for "fresh image"
  - compare hash to downloaded manifest file for "currently booted image" (this is super quick)

#### step b) do the update:
- while-loop going over each block of manifest file for "fresh image"
  - search for the hash in "currently booted image" manifest
  - if hash is found -> copy block over from "currently booted image" into the "stale image" (DM-Verity will make fail that if the "currently booted image" is corrupted)
  - if hashes don't match, download the block into tmp dir using HTTP range request and write block to "stale image"
  - need to do a read-back to check the hash, this should also force the actual write

- after the while loop ran completely, we mark the "stale image" as next image to boot

#### possible optimizations:
- mostly same as in Approach 2)

#### downsides:
- must use the "currently booted image" as source for copying, can't use "most recent installed image" anymore
- download size can end up larger than what we tell the user if there are corrupted blocks

#### upsides:
- checking download size is super fast


## Questions:
- can we use the hash format of DM-Verity itself and are we able to read the hash data of DM-Verity somehow
  - and do we want to lock our stuff to their hash format? if they change the format, we have to change ours!
- should we cache hashes somehow to make update checks faster?
- apparently some CDNs don't support putting multiple ranges into a single Range header
  - need to look into how many that actually are
- which hashing algorithm to use?
