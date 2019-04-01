`bitcoin-iterate` is a command-line tool which quickly iterates over
`bitcoind` block files and prints data about them.

# Installation

Clone this repository:

```
$ git clone https://github.com/unchained-capital/bitcoin-iterate
$ cd bitcoin-iterate
```

Install necessary libraries:

```
$ sudo apt-get install libssl-dev
```


Compile  the `bitcoin-iterate` program:

```
$ make
```

Now see if you can display the program's help output:

```
$ ./bitcoin-iterate --help
```

You’ll see several general options, but the most important are

* `--blockdir` is the directory containing Bitcoin’s blocks (ensure
  that `bitcoin-iterate` can read the contents of this directory).
  Defaults to `${HOME}/.bitcoin/blocks` (or
  `${HOME}/.bitcoin/testnet3/blocks` if using testnet).

* `--start` (or `--start-hash`) tell `bitcoin-iterate` which block
  height (or hash) to begin iteration from. The default behavior is to
  start from the genesis block at the beginning of the blockchain.

* `--end` (or `--end-hash`) tell `bitcoin-iterate` which block height
  (or hash) to end iteration at. The default behavior is to end at the
  last known block in the blockchain.

* `--cache` is a directory in which to write temporary data on the
  first iteration which speeds up subsequent iterations (ensure that
  `bitcoin-iterate` can write to this directory).

## Dependencies

This repository does not include OpenSSL which is required to compile
and run the `bitcoin-iterate` program.  Ensure you have installed
OpenSSL.

If you have a custom location for OpenSSL (e.g. - you are compiling on
a more recent Macbook), you can pass the appropriate include directory
via:

```
$ make -e COMPILER_FLAGS='-I /path/to/your/openssl/includes'
```

# Usage

`bitcoin-iterate` begins iteration at the starting block and continues
until the ending block. For each block in between (inclusive),
`bitcoin-iterate` will iterate over:

* the block metadata
* each transaction in order of appearance.  For each transaction, `bitcoin-iterate` will iterate over 
  * each input in order
  * each output in order
	* each UTXO in the UTXO set (optional)

Each data structure (block, transaction, input, output, UTXO)
corresponds a command-line flag which allows you to specify a format
string containing format codes (e.g. - `%bN` for block height) which
will be replaced by the corresponding values from the data structure
and printed to `STDOUT` during iteration.

* The format string in `--block` will be used on each block, allowing
  you to print out the height, hash, timestamp, number of
  transactions, &c.

* The format string in `--transaction` will be used on each
  transaction, allowing you to print out the hash, number of inputs or
  outputs, amounts, or even the full transaction data.

* The format strings in `--input` and `--output` will be used on each
  transaction input and output, allowing you to print out hashes,
  amounts, or even scripts.

* The format string in `--utxo` will be used on each UTXO present in
  the UTXO set of the block, allowing you to print out amounts,
  timestamps, &c. Iterating over UTXOs is time consuming, so by
  default `bitcoin-iterate` will not iterate over the UTXO set unless
  the `--utxo` flag is present.  Even then, the default behavior is to
  iterate over the UTXO set every 144 blocks (approximately once per
  day). You can use the `--utxo-period` flag to change this frequency.

To see all the format code options, run `.bitcoin-iterate --help`.

Using `bitcoin-iterate` amounts to deciding which blocks you want to
iterate between and specifying which format strings you want to use to
extract data. Here is a simple test which prints out the block numbers
and hashes of the first 10 blocks:

```
$ ./bitcoin-iterate --end 10 --block "%bN %bh"
...
0 6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000
1 4860eb18bf1b1620e37e9490fc8a427514416fd75159ab86688e9a8300000000
2 bddd99ccfda39da1b108ce1a5d70038d0a967bacb68b6b63065f626a00000000
3 4944469562ae1c2c74d9a535e00b6f3e40ffbad4f2fda3895501b58200000000
4 85144a84488ea88d221c8bd6c059da090e88f8a2c99690ee55dbba4e00000000
5 fc33f596f822a0a1951ffdbf2a897b095636ad871707bf5d3162729b00000000
6 8d778fdc15a2d3fb76b7122a3b5582bea4f21f5a0c693537e7a0313000000000
7 4494c8cf4154bdcc0720cd4a59d9c9b285e4b146d45f061d2b6c967100000000
8 c60ddef1b7618ca2348a46e868afc26e3efc68226c78aa47f8488c4000000000
9 0508085c47cc849eb80ea905cc7800a3be674ffc57263cf210c59d8d00000000
10 e915d9a478e3adf3186c07c61a22228b10fd87df343c92782ecc052c00000000
```

See the "Examples" section below for more examples.

## Utilizing Cache

### Block Cache

`bitcoind` does not store the blocks in the blockchain in order on
disk.  This is because it cannot know when first receiving a block
whether it will be "uncled" (replaced by another sequence of blocks on
a longer chain).  Each block on disk has a unique hash identifying it
as well as the hash of the previous block it follows.

As a result, when `bitcoin-iterate` runs for the first time, it has to
read through all `bitcoind` block files on disk to determine which
blocks are meant to be in which order. It writes this order (as well
as the correspondence between blocks and offsets in block files on
disk) to the directory you specify with the `--cache` option. If you
don’t provide a cache directory, `bitcoin-iterate` has to repeat this
bookkeeping work on each run, so having a cache greatly increases the
speed of `bitcoin-iterate` starting up on subsequent
invocations. 

**Note:** If the last modified timestamp of the latest `bitcoind`
block file is newer than the timestamp of the `bitcoin-iterate` block
cache file , the block cache is considered invalid, and
`bitcoin-iterate` will need to build it again.  For this reason, if
you're doing an extended analysis session, it may be best to use a
static copy of the `bitcoind` block files that aren't being actively
appended to by `bitcoind`.  Or just turn off `bitcoind` itself so the
files don't change.

### UTXO Cache

Blocks, transactions, inputs, and outputs are all written by
`bitcoind` to disk in the block files scanned by
`bitcoin-iterate`. But the UTXO set for a block -- the set of all
unspent transaction outputs from prior transactions on the chain -- is
**not** written to disk because the UTXO set is a *consequence* of the
data on disk: at a given block, the history of all prior transactions
uniquely determines the UTXO set at that block. So, `bitcoind` does
not write the UTXO to disk, but keeps it in memory. As transactions
get mined into blocks, `bitcoind` updates its in-memory set of UTXOs,
deleting any which were consumed by the transaction and adding more
which the transaction itself produced.

As a result, when `bitcoin-iterate` is asked to iterate over the UTXO
set at a particular block, it must calculate that UTXO set based on
the UTXO set of the prior block.  This process goes “backwards” all
the way to the genesis block, which created the first (technically
unspendable) UTXO. The first time `bitcoin-iterate` is invoked with a
particular starting block it starts from the genesis block and
calculates the UTXO set at that block, and then saves it to the cache
directory. When subsequently iterating over UTXOs and *starting from
the same block*, `bitcoin-iterate` can simply use the pre-calculated
UTXO set it has cached on disk. If iterating over UTXOs and starting
from a *different* block, `bitcoin-iterate` has to calculate UTXOs
again from scratch and save a new UTXO cache file.

## Directly using the C API

This repository also defines a C function `iterate` which you can use
in your own programs to specify custom functions to run on each data
structure (block, transaction, input, output, or UTXO).  The
`bitcoin-iterate` program uses this `iterate` API internally.

Writing your own C programs against the `iterate` API is useful if you
have significant processing to do on each data structure and you want
it to happen inside C where it will be fast with minimal overhead
between the data in the block files on disk and your own custom code.
It's also very useful if you intend to repeateadly process UTXO sets
which can grow very large, more than 1 GB per block, and are therefore
sometimes easier to process during iteration than via intermediate
files on disk.

The example C code below (`utxo_summarizer.c`) defines a custom
function to be run on each UTXO and passes this function into the
`iterate` API:

```c
/* utxo_summarizer.c

This code defines a custom function `accumulate_utxo` which is used to
count the number of UTXOs and the total unspent amount of all UTXOs at
each block.

*/

// C standard libraries
#include <stdio.h>
#include <inttypes.h>

// From bitcoin-iterate repository
#include "iterate.h"

/* Iterator */
static struct block *last_block;

/* UTXOs */
static size_t utxo_count_block = 0;
static s64    utxo_amount_block = 0;

/* This function will be called at the end of each block, just before
iteration proceeds to the following block. */
static void transition_to_new_block(const struct utxo_map *utxo_map, struct block *new_block) {  
  if (last_block) {
    printf("%ui %zui %"PRIi64"i\n", last_block->height, utxo_count_block, utxo_amount_block);
  }
  utxo_count_block  = 0;
  utxo_amount_block = 0;
}

/* This function will be called on each UTXO. */
static void accumulate_utxo(const struct utxo_map *utxo_map, struct block *current_block, struct block *last_utxo_block, struct utxo *u) {
  int i;
  utxo_count_block  += utxo->unspent_outputs;
  utxo_amount_block += utxo->unspent;
}

int main(int argc, char *argv[]) {
  iterate(
          /* Directories & network to use. */
          NULL,                         // blockdir -- NULL means use default
          NULL,                         // cache -- NULL means no cache
          false,                        // use testnet?

	      /* Iterate between genesis block (height: 0) and whichever
          is the final block on disk (height: -1). */
          0,                            // starting block height
          -1,                           // ending block height

          /* We've already specified starting/ending block heights, so
          no need to pass starting/ending block hashes; hence the null
          byte string here. */
          { 0 },                        // starting block hash
          { 0 },                        // ending block hash

          /* What to do about UTXOs? */
          true,                         // Should we be iterating over UTXOs?
          144,                          // UTXO period to use when iterating (144 ~ 1x per day).

          /* Some general options. */
          true,                         // use mmap to process blockfiles
          0,                            // print progress marks while iterating
          false,                        // whether to silence debugging output

          /* Functions to call when iterating over each data
          structure. */
          transition_to_new_block,      // called on each block
          NULL,                         // called on each transaction
          NULL,                         // called on each input
          NULL,                         // called on each output
          accumulate_utxo               // called on each UTXO
  );
  return 0;
}
```
You can compile and run this program:

```
$ cc -I ccan/ ... utxo_summarizer.c
$ ...
$ ./utxo_summarizer
```

## Miscellanous Usage Notes

There are a few usage gotchas and notes worth documenting here:

* **Hashes are little-endian.** When you use formatting flags such as
  `%bh` or `%th` to print out block & transaction hashes, you will see
  the output as
  [little-endian](https://en.wikipedia.org/wiki/Endianness).  Block
  explorers and similar tools usually convert these hashes to
  big-endian for display.  If you are comparing between your output
  and what you see online, don't forget to flip the endianness!

# Examples

The examples below assume you are running on mainnet (not testnet),
`bitcoind` block files are in the default location
(`${HOME}/.bitcoin/blocks`), and you are not using a cache.  The
examples will be much faster if you provide a `--cache` directory.

* Show block hash and transaction size for each transaction in the
  main chain:

  ```
  $ ./bitcoin-iterate -q --tx=%bh,%tl
  ...
  6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000,204
  4860eb18bf1b1620e37e9490fc8a427514416fd75159ab86688e9a8300000000,134
  bddd99ccfda39da1b108ce1a5d70038d0a967bacb68b6b63065f626a00000000,134
  4944469562ae1c2c74d9a535e00b6f3e40ffbad4f2fda3895501b58200000000,134
  85144a84488ea88d221c8bd6c059da090e88f8a2c99690ee55dbba4e00000000,134
  ...
  ```

* Show the five largest blocks, by height and blockhash:

  ```
  $ ./bitcoin-iterate -q --block='%bl %bN %bh' | sort -nr | head -n5
  ...
  999993 343188 96395be50a116886fb04a1262aa89aaa765c4af524d0ae090000000000000000
  999991 355770 5ce1f5e58d5d718374b1bdecb474eae5cdd91e1b8a3256050000000000000000
  999990 327145 821196665ed237f53ca2c7eafe05c97d26743c07d2b539110000000000000000
  999989 345578 0294fa5b68babd848a1fe097b075a856bfd9eb9a619efd130000000000000000
  999989 333434 f9776021575b6e408d7745d115db75e17937fc40ab9f0e190000000000000000
  ```

* Show output script sizes in the main chain (block number,
  transaction number, output number, output script length):

  ```
  $ ./bitcoin-iterate -q --output=%bN,%tN,%oN:%ol
  ...
  0,0,0:67
  1,0,0:67
  2,0,0:67
  3,0,0:67
  4,0,0:67
  ...
  ```

You can see some examples by looking at the [manual page source](https://github.com/rustyrussell/bitcoin-iterate/blob/master/doc/bitcoin-iterate.1.txt).

## Enhancements

Happy to consider them!

You can reach me on IRC (rusty on #bitcoin-wizards on Freenode), and
of course, via pull requests and the
[Github bug tracker](https://github.com/rustyrussell/bitcoin-iterate/issues).

Good luck!<br>
Rusty Russell.
