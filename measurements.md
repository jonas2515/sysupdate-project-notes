# measurements to confirm the approach of making deltas:

## comparing delta of /usr partitions of minor upgrade: two nightlies from two following days
- images we look at
  - from: gnome_os_941356-x86_64.iso
    - timestamp: 2025-11-03T20:00:28.000Z
  - to: gnome_os_941676-x86_64.iso
    - timestamp: 2025-11-04T12:08:31.000Z

- Adrian got a "theoretical best case delta" for these images:
    the only thing that changed between them is the introduction of components/protobuf.c, which introduces 9.8 MiB worth of (uncompresed) content to the image. After the compression, the new image is 5 MiB larger than the old one. Since the addition of that package is the only thing that changed, that's theoretically the best case our delta can have
  - after checking again, it seems there is a bit more: the protobuf libraries + executables are added, the manifest.json differs, and various efi binaries in /mnt/share/factory/efi differ, together it's 15 MiB

- GNOME OS upstream images are generated with "mkfs.erofs -z zstd -E all-fragments,fragdedupe=inode"
  - since commit https://gitlab.gnome.org/GNOME/gnome-build-meta/-/commit/86e48cedaeddf9ee4eaae95fdad1c9e032443f00, they are generated with: -z zstd -E fragments,fragdedupe=inode

- we're doing our hashes with a 4 KiB (4096 bytes) hash-block size

### comparison results

- upstream image (-z zstd -E all-fragments,fragdedupe=inode)
  - result: n_blocks_matching=262241, n_blocks_not_matching=381695, percent_matching=0.407247
            n_blocks_at_same_position=16616, percent_in_same_position=0.025803806
            total_bytes_read_image_from=2631630848 (2509 MiB), total_bytes_read_image_to=2637561856 (2515 MiB)
            n_blocks_image_from=642488 (2509 MiB), n_blocks_image_to=643936 (2515 MiB)

  - with desync according to Adrian: checking the upstream images with desync: 44% shared

#### rebuilding the erofs and trying different options ourselves

- did all these tests with versions
  - kernel: 6.16.8-400.asahi.fc43.aarch64+16k
  - mkfs.erofs 1.8.10

- no options at all:
  - result: n_blocks_matching=1038941, n_blocks_not_matching=48691, percent_matching=0.9552321
            n_blocks_at_same_position=20849, percent_in_same_position=0.019169167
            total_bytes_read_image_from=5453398016 (5200 MiB), total_bytes_read_image_to=5463748608 (5210 MiB)
            n_blocks_image_from=1085114 (4238 MiB), n_blocks_image_to=1087632 (4248 MiB)

  - with desync according to Adrian: OK so I generated a basic erofs image, with no compression or anything. Just fully default invocation of mkfs.erofs. With desync's standard chunk size it shares 92% of the blocks in the image


- -E all-fragments
  - result: n_blocks_matching=1038941, n_blocks_not_matching=48691, percent_matching=0.9552321
            n_blocks_at_same_position=20849, percent_in_same_position=0.019169167
            total_bytes_read_image_from=5453398016 (5200 MiB), total_bytes_read_image_to=5463748608 (5210 MiB)
            n_blocks_image_from=1085114 (4238 MiB), n_blocks_image_to=1087632 (4248 MiB)


- -E all-fragments,fragdedupe=inode
  - result: n_blocks_matching=1038941, n_blocks_not_matching=48691, percent_matching=0.9552321
            n_blocks_at_same_position=20849, percent_in_same_position=0.019169167
            total_bytes_read_image_from=5453398016 (5200 MiB), total_bytes_read_image_to=5463748608 (5210 MiB)
            n_blocks_image_from=1085114 (4238 MiB), n_blocks_image_to=1087632 (4248 MiB)


- -E fragments
  - result: n_blocks_matching=1038941, n_blocks_not_matching=48691, percent_matching=0.9552321
            n_blocks_at_same_position=20849, percent_in_same_position=0.019169167
            total_bytes_read_image_from=5453398016 (5200 MiB), total_bytes_read_image_to=5463748608 (5210 MiB)
            n_blocks_image_from=1085114 (4238 MiB), n_blocks_image_to=1087632 (4248 MiB)


- -E fragments,fragdedupe=inode
  - result: n_blocks_matching=1038941, n_blocks_not_matching=48691, percent_matching=0.9552321
            n_blocks_at_same_position=20849, percent_in_same_position=0.019169167
            total_bytes_read_image_from=5453398016 (5200 MiB), total_bytes_read_image_to=5463748608 (5210 MiB)
            n_blocks_image_from=1085114 (4238 MiB), n_blocks_image_to=1087632 (4248 MiB)


- -z zstd -E all-fragments
  - result: n_blocks_matching=287734, n_blocks_not_matching=353236, percent_matching=0.448904
            n_blocks_at_same_position=26869, percent_in_same_position=0.04191928
            total_bytes_read_image_from=2712621056 (2586 MiB), total_bytes_read_image_to=2717491200 (2591 MiB)
            n_blocks_image_from=639681 (2498 MiB), n_blocks_image_to=640970 (2503 MiB)


- -z zstd -E all-fragments,fragdedupe=inode
  - result: n_blocks_matching=262241, n_blocks_not_matching=382700, percent_matching=0.4066124
            n_blocks_at_same_position=16616, percent_in_same_position=0.025763597
            total_bytes_read_image_from=2635747328 (2513 MiB), total_bytes_read_image_to=2641678336 (2519 MiB)
            n_blocks_image_from=643493 (2513 MiB), n_blocks_image_to=644941 (2519 MiB)


- -z zstd -E fragments
  - result: n_blocks_matching=623466, n_blocks_not_matching=14361, percent_matching=0.97748446
            n_blocks_at_same_position=35034, percent_in_same_position=0.054927118
            total_bytes_read_image_from=2607054848 (2486 MiB), total_bytes_read_image_to=2612539392 (2491 MiB)
            n_blocks_image_from=636488 (2486 MiB), n_blocks_image_to=637827 (2491 MiB)


  - with desync according to Adrian: OK, desync with its default configuration with -E fragments -z zstd will share only 93% of the blocks resulting in 170 MiB deltas, ~3x the size of the trivial 4k block thing
  - Adrian also tried forcing desync to 4 KiB chunk size: desync blows up if you tell it to use 4k blocks, but if you tell it to use minimum 4k blocks with an average size of 4k and a maximum size of 5k it does work. Resulting in 94% of blocks shared and a delta of 148 MiB

- -z zstd -E fragments,fragdedupe=inode
  - result: n_blocks_matching=623466, n_blocks_not_matching=14361, percent_matching=0.97748446
            n_blocks_at_same_position=35034, percent_in_same_position=0.054927118
            total_bytes_read_image_from=2607054848 (2486 MiB), total_bytes_read_image_to=2612539392 (2491 MiB)
            n_blocks_image_from=636488 (2486 MiB), n_blocks_image_to=637827 (2491 MiB)


- 16 KiB hash-block size (but with 4 KiB block size in erofs): -z zstd -E fragments
  - result: n_blocks_matching=10271, n_blocks_not_matching=149186, percent_matching=0.06441235
            n_blocks_at_same_position=8756, percent_in_same_position=0.054911356
            total_bytes_read_image_from=2607054848 (2486 MiB), total_bytes_read_image_to=2612539392 (2491 MiB)
            n_blocks_image_from=159122 (2486 MiB), n_blocks_image_to=159457 (2491 MiB)

- 512 byte hash-block size (but with 4 KiB block size in erofs): -z zstd -E fragments
  - result: n_blocks_matching=4961118, n_blocks_not_matching=110075, percent_matching=0.9782941
            n_blocks_at_same_position=279295, percent_in_same_position=0.05507481
            total_bytes_read_image_from=2607054848 (2486 MiB), total_bytes_read_image_to=2612539392 (2491 MiB)
            n_blocks_image_from=5060485 (2470 MiB), n_blocks_image_to=5071193 (2476 MiB)

- 512 byte hash-block size (with 512 byte block size in erofs): -z zstd -E fragments
  - result: n_blocks_matching=4977223, n_blocks_not_matching=164910, percent_matching=0.96792966
            n_blocks_at_same_position=309, percent_in_same_position=0.000060091796
            total_bytes_read_image_from=2644567040 (2522 MiB), total_bytes_read_image_to=2650079744 (2527 MiB)
            n_blocks_image_from=5131368 (2505 MiB), n_blocks_image_to=5142133 (2510 MiB)


#### what does that mean?

- with compression enabled, we get best results with "-z zstd -E fragments", adding option "fragdedupe=inode" doesn't seem to change anything about the image
- with 0.97748446 percent matching, we can ship a delta of n_blocks_not_matching=14361 blocks, meaning ~56.1 MiB
  - need to add the size of the sha256 file with the block hashes to the amount of data shipped: 32 bytes * 637827 = ~19.5 MiB
  - that would make a total download size for the update of roughly 76 MiB

- another observation by Adrian: Also in terms of index: desync's index file is 20 MiB in size, when using a 4k chunk size. With a larger chunk size (like the 16k-minimum 256k-maximum picked by desync by default) the index becomes as small as 1.5 MiB
  - that would be a total download size of 170 MiB + 1.5 MiB, so roughly 171 MiB
  - for dsync with the 4k block size, it would be 148 MiB + 20 MiB, so in total 168 MiB

- reading and computing hashes from the zstd compressed image with 4 KiB hash-block size takes 2:09 minutes on my (fairly fast) machine
  - that's not nothing, esp given that it takes 100% of a cpu core
  - shouldn't need the cryptographic security of sha256 when dm-verity is already there, so when using a fast hashing algorithm, we can make this a lot faster


## comparing delta of /usr partitions of major upgrade: GNOME 48 to GNOME 49
- images we look at:
  - from: gnome-48/sysupdate/usr-arm64_gnome_48.913249_c1648fcd-dc5c-5f8e-aac8-5e7d18569253.raw.xz
    - timestamp: 2025-09-18T23:58:22.000Z
  - to: gnome-49/sysupdate/usr-arm64_gnome_49.912813_ceab62e7-47c9-d644-49ae-63fdc839f0a8.raw
    - timestamp: 2025-09-18T09:58:04.000Z

- images were squashfs, so had to rebuild them to erofs

#### rebuilding the erofs

- did tests with versions
  - kernel: 6.16.8-400.asahi.fc43.aarch64+16k
  - mkfs.erofs 1.8.10

- -z zstd -E fragments
  - result: n_blocks_matching=331044, n_blocks_not_matching=299144, percent_matching=0.5253099
            n_blocks_at_same_position=0, percent_in_same_position=0
            total_bytes_read_image_from=2550554624 (2432 MiB), total_bytes_read_image_to=2581250048 (2461 MiB)
            n_blocks_image_from=622694 (2432 MiB), n_blocks_image_to=630188 (2461 MiB)


- 512 byte hash-block size (but with 4 KiB block size in erofs): -z zstd -E fragments
  - result: n_blocks_matching=2634829, n_blocks_not_matching=2374935, percent_matching=0.52593875
            n_blocks_at_same_position=0, percent_in_same_position=0
            total_bytes_read_image_from=2550554624 (2432 MiB), total_bytes_read_image_to=2581250048 (2461 MiB)
            n_blocks_image_from=4949589 (2416 MiB), n_blocks_image_to=5009764 (2446 MiB)

#### what does that mean?

- looks like a pretty good result actually
