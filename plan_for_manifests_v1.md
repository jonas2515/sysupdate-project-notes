## ideas for manifest file

- use a fast, and less space intesive non-cryptographic hashing function for the block hashing
  - we don't need cryptographic hashing of blocks as long as we verify the image integrity after downloading
    - first of all dm-verity will ensure this for us, but only at runtime, and with very bad error handling
    - and we can also just ship *a single* cryptographically secure hash with our manifest that we verify the image with after writing to disk

- for the manifest file, we need to store:
  - a version of the file format (because things about the manifest and delta mechanism might change)
  - the cryptographically secure hash of the whole image
  - all the smaller hashes for the individual blocks
    - those will need to be in an efficient, tightly packed file format

- lets use gvariant for the manifest file
  - we can describe the version and cryptographically secure hash with it
  - we can efficiently put an array with fixed size elements into it (for the block hashes)
  - there's a rust impl: https://docs.rs/gvariant/latest/gvariant/
  - using C struct notation, our manifest could look like this:
struct manifest {
  uint32_t version;         /* manifest version */
  uint8_t salt[32];         /* sha256 salt */
  uint8_t secure_hash[32] ; /* sha256 secure hash */
  uint64_t hash_blocks[];   /* block hashes */
}


- we need to create the dm-verity partition, too
  - some docs on dmVerity: https://gitlab.com/cryptsetup/cryptsetup/-/wikis/DMVerity
  - https://man7.org/linux/man-pages/man8/veritysetup.8.html
  - the hash tree format works like this
    - it's always at least 0x2000 bytes long
      - length of file seems rounded up to next 0x1000 offset
    - from looking at the thing, it looks like the hashes are starting at offset 0x1000
    - if we have more than 1 hash block, hash-block hashes are at 0x1000, and next layer of block hashes at (rounded up to next 0x1000 offset)
    - it seems like for 128 hashes, there's one hash-block layer created, and then for 128 of these, there's another hash-block layer created
    - number of hash blocks in veritysetup output is calculated like this nHashBlocksLayer1 + nHashBlocksLayer2 + nHashBlocksLayer3 + nHashBlocksLayerN + 1
    - so format is like this for say 2 layers of hash blocks
      0x0000 header
      0x1000 layer2, hash1
      0x1020 layer2, hash2
      0x2000 layer1, hash1
      0x2020 layer1, hash2
      0x2040 layer1, hash3
      ...
      0x3000 data-blocks
      0x

- do we really want to deal with the verity partition ourselves?
  - crypto stuff, probably a good idea to let veritysetup do the job instead
  - also there's a salt involved
    - first of all would be good to know why?
      - https://unix.stackexchange.com/questions/264490/why-does-dm-verity-use-a-salt
    - that means we can't know the root hash beforehand, unless we use a fixed salt
      - using a fixed salt would defeat the point, but we can use a predefined salt for each new manifest/OS-image
      - predefined salt is a little bit weaker than generating the salt on each machine individually
        - but we can't generate the salt locally on the machine, because that would change the root hash
          (and root hash is part of kernel cmdline -> that's signed by the OS vendor (us))
        - still a good tradeoff, because in return we get working secure boot, not just measured boot
  - so
    - we download the image, then do a "normal" sha256 hash of the whole image and compare it with the one from the manifest...
    - then we call veritysetup to create the verity partition (with the given salt from the manifest)
    - OR: use veritysetup immediately and then just compare the root hash (we have to precompute that one on the build server anyway, since it needs to be shipped with the kernel cmdline)
    -> if integrity doesn't match, restart the whole process, but this time download the full image (no delta updater) (likely there's a crc64 collision somewhere, and in that case using delta updater again won't help)

- looking into hashing algorithms:
  - default hashing function of rust is not guaranteed to be the same and hashes shouldn't be used between multiple versions of rust
  - could just go with good old crc32/crc64: https://crates.io/crates/crc-fast
    - crc is a well known algorithm and will remain the same
    - if library goes unmaintained, easy to swap out with something maintained
    - only 32/64 bits, so manifest would be like 2.5 or 5 MiB in size, perfect!
    - we have about 600.000 blocks to hash, here's a table with collision chances: https://preshing.com/20110504/hash-collision-probabilities/
      - for 32 bit hash length, the collision is extremely likely, it's 1 out of 2 for 70k hashes already!
      - for 64 bit hash length, collision chance is 1 in 100 million for 600k hashes, that seems low enough for us
      -> we need 64 bits hash for sure, let's go with Crc64-Nvme
  - other, more custom hashing algorithms would be https://crates.io/crates/foldhash, https://crates.io/crates/rustc-hash


- optimization idea with minimal extra work on the server:
  - right now we still have 5+5MiB (old + new) of manifest downloads for a single update
  - between updates, a lot of blocks may stay the same, but their positions change (like 95% of them)
    - so if we pre-generate the deltas between manifests on the server already, we would still end up with like 95% of the hashes necessary
    - but: we might as well leave out the hashes now and just show which blocks need to be downloaded and from->to where blocks are moving
      - with a 2.4GiB file, we have about 600k 4096 byte blocks
      - block position in the new image can a uint32, or even a uint24
      - and block position in the existing image can be expressed as the offset in the mapping file
        - lots of well compressible empty space in the mapping file
      - with uint24, that would be 1.7 MiB max size for a mapping file
        - likely even less with compression!!
        - should experiment a bit with this, but seems promising

- when reading the manifest, need to do the version check in a forward-compatible way
  - version check should only read the first 4 bytes (if the version check is part of normal parsing, it might fail as soon as the file format changes)
