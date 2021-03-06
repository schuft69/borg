.. include:: ../global.rst.inc
.. highlight:: none

.. _data-structures:

Data structures and file formats
================================

.. _repository:

Repository
----------

.. Some parts of this description were taken from the Repository docstring

|project_name| stores its data in a `Repository`, which is a filesystem-based
transactional key-value store. Thus the repository does not know about
the concept of archives or items.

Each repository has the following file structure:

README
  simple text file telling that this is a |project_name| repository

config
  repository configuration

data/
  directory where the actual data is stored

hints.%d
  hints for repository compaction

index.%d
  repository index

lock.roster and lock.exclusive/*
  used by the locking system to manage shared and exclusive locks

Transactionality is achieved by using a log (aka journal) to record changes. The log is a series of numbered files
called segments_. Each segment is a series of log entries. The segment number together with the offset of each
entry relative to its segment start establishes an ordering of the log entries. This is the "definition" of
time for the purposes of the log.

.. _config-file:

Config file
~~~~~~~~~~~

Each repository has a ``config`` file which which is a ``INI``-style file
and looks like this::

    [repository]
    version = 1
    segments_per_dir = 10000
    max_segment_size = 5242880
    id = 57d6c1d52ce76a836b532b0e42e677dec6af9fca3673db511279358828a21ed6

This is where the ``repository.id`` is stored. It is a unique
identifier for repositories. It will not change if you move the
repository around so you can make a local transfer then decide to move
the repository to another (even remote) location at a later time.

Keys
~~~~

Repository keys are byte-strings of fixed length (32 bytes), they
don't have a particular meaning (except for the Manifest_).

Normally the keys are computed like this::

  key = id = id_hash(unencrypted_data)

The id_hash function depends on the :ref:`encryption mode <borg_init>`.

As the id / key is used for deduplication, id_hash must be a cryptographically
strong hash or MAC.

Segments
~~~~~~~~

A |project_name| repository is a filesystem based transactional key/value
store. It makes extensive use of msgpack_ to store data and, unless
otherwise noted, data is stored in msgpack_ encoded files.

Objects referenced by a key are stored inline in files (`segments`) of approx.
500 MB size in numbered subdirectories of ``repo/data``.

A segment starts with a magic number (``BORG_SEG`` as an eight byte ASCII string),
followed by a number of log entries. Each log entry consists of:

* size of the entry
* CRC32 of the entire entry (for a PUT this includes the data)
* entry tag: PUT, DELETE or COMMIT
* PUT and DELETE follow this with the 32 byte key
* PUT follow the key with the data

Those files are strictly append-only and modified only once.

Tag is either ``PUT``, ``DELETE``, or ``COMMIT``.

When an object is written to the repository a ``PUT`` entry is written
to the file containing the object id and data. If an object is deleted
a ``DELETE`` entry is appended with the object id.

A ``COMMIT`` tag is written when a repository transaction is
committed.

When a repository is opened any ``PUT`` or ``DELETE`` operations not
followed by a ``COMMIT`` tag are discarded since they are part of a
partial/uncommitted transaction.

Compaction
~~~~~~~~~~

For a given key only the last entry regarding the key, which is called current (all other entries are called
superseded), is relevant: If there is no entry or the last entry is a DELETE then the key does not exist.
Otherwise the last PUT defines the value of the key.

By superseding a PUT (with either another PUT or a DELETE) the log entry becomes obsolete. A segment containing
such obsolete entries is called sparse, while a segment containing no such entries is called compact.

Since writing a ``DELETE`` tag does not actually delete any data and
thus does not free disk space any log-based data store will need a
compaction strategy.

Borg tracks which segments are sparse and does a forward compaction
when a commit is issued (unless the :ref:`append_only_mode` is
active).

Compaction processes sparse segments from oldest to newest; sparse segments
which don't contain enough deleted data to justify compaction are skipped. This
avoids doing e.g. 500 MB of writing current data to a new segment when only
a couple kB were deleted in a segment.

Segments that are compacted are read in entirety. Current entries are written to
a new segment, while superseded entries are omitted. After each segment an intermediary
commit is written to the new segment, data is synced and the old segment is deleted --
freeing disk space.

(The actual algorithm is more complex to avoid various consistency issues, refer to
the ``borg.repository`` module for more comments and documentation on these issues.)

.. _manifest:

The manifest
------------

The manifest is an object with an all-zero key that references all the
archives. It contains:

* Manifest version
* A list of archive infos
* timestamp
* config

Each archive info contains:

* name
* id
* time

It is the last object stored, in the last segment, and is replaced
each time an archive is added, modified or deleted.

.. _archive:

Archives
--------

The archive metadata does not contain the file items directly. Only
references to other objects that contain that data. An archive is an
object that contains:

* version
* name
* list of chunks containing item metadata (size: count * ~40B)
* cmdline
* hostname
* username
* time

.. _archive_limitation:

Note about archive limitations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The archive is currently stored as a single object in the repository
and thus limited in size to MAX_OBJECT_SIZE (20MiB).

As one chunk list entry is ~40B, that means we can reference ~500.000 item
metadata stream chunks per archive.

Each item metadata stream chunk is ~128kiB (see hardcoded ITEMS_CHUNKER_PARAMS).

So that means the whole item metadata stream is limited to ~64GiB chunks.
If compression is used, the amount of storable metadata is bigger - by the
compression factor.

If the medium size of an item entry is 100B (small size file, no ACLs/xattrs),
that means a limit of ~640 million files/directories per archive.

If the medium size of an item entry is 2kB (~100MB size files or more
ACLs/xattrs), the limit will be ~32 million files/directories per archive.

If one tries to create an archive object bigger than MAX_OBJECT_SIZE, a fatal
IntegrityError will be raised.

A workaround is to create multiple archives with less items each, see
also :issue:`1452`.

.. _item:

Items
-----

Each item represents a file, directory or other fs item and is stored as an
``item`` dictionary that contains:

* path
* list of data chunks (size: count * ~40B)
* user
* group
* uid
* gid
* mode (item type + permissions)
* source (for links)
* rdev (for devices)
* mtime, atime, ctime in nanoseconds
* xattrs
* acl
* bsdfiles

All items are serialized using msgpack and the resulting byte stream
is fed into the same chunker algorithm as used for regular file data
and turned into deduplicated chunks. The reference to these chunks is then added
to the archive metadata. To achieve a finer granularity on this metadata
stream, we use different chunker params for this chunker, which result in
smaller chunks.

A chunk is stored as an object as well, of course.

.. _chunks:
.. _chunker_details:

Chunks
------

The |project_name| chunker uses a rolling hash computed by the Buzhash_ algorithm.
It triggers (chunks) when the last HASH_MASK_BITS bits of the hash are zero,
producing chunks of 2^HASH_MASK_BITS Bytes on average.

Buzhash is **only** used for cutting the chunks at places defined by the
content, the buzhash value is **not** used as the deduplication criteria (we
use a cryptographically strong hash/MAC over the chunk contents for this, the
id_hash).

``borg create --chunker-params CHUNK_MIN_EXP,CHUNK_MAX_EXP,HASH_MASK_BITS,HASH_WINDOW_SIZE``
can be used to tune the chunker parameters, the default is:

- CHUNK_MIN_EXP = 19 (minimum chunk size = 2^19 B = 512 kiB)
- CHUNK_MAX_EXP = 23 (maximum chunk size = 2^23 B = 8 MiB)
- HASH_MASK_BITS = 21 (statistical medium chunk size ~= 2^21 B = 2 MiB)
- HASH_WINDOW_SIZE = 4095 [B] (`0xFFF`)

The buzhash table is altered by XORing it with a seed randomly generated once
for the archive, and stored encrypted in the keyfile. This is to prevent chunk
size based fingerprinting attacks on your encrypted repo contents (to guess
what files you have based on a specific set of chunk sizes).

For some more general usage hints see also ``--chunker-params``.

.. _cache:

Indexes / Caches
----------------

The **files cache** is stored in ``cache/files`` and is used at backup time to
quickly determine whether a given file is unchanged and we have all its chunks.

The files cache is a key -> value mapping and contains:

* key:

  - full, absolute file path id_hash
* value:

  - file inode number
  - file size
  - file mtime_ns
  - list of file content chunk id hashes
  - age (0 [newest], 1, 2, 3, ..., BORG_FILES_CACHE_TTL - 1)

To determine whether a file has not changed, cached values are looked up via
the key in the mapping and compared to the current file attribute values.

If the file's size, mtime_ns and inode number is still the same, it is
considered to not have changed. In that case, we check that all file content
chunks are (still) present in the repository (we check that via the chunks
cache).

If everything is matching and all chunks are present, the file is not read /
chunked / hashed again (but still a file metadata item is written to the
archive, made from fresh file metadata read from the filesystem). This is
what makes borg so fast when processing unchanged files.

If there is a mismatch or a chunk is missing, the file is read / chunked /
hashed. Chunks already present in repo won't be transferred to repo again.

The inode number is stored and compared to make sure we distinguish between
different files, as a single path may not be unique across different
archives in different setups.

Not all filesystems have stable inode numbers. If that is the case, borg can
be told to ignore the inode number in the check via --ignore-inode.

The age value is used for cache management. If a file is "seen" in a backup
run, its age is reset to 0, otherwise its age is incremented by one.
If a file was not seen in BORG_FILES_CACHE_TTL backups, its cache entry is
removed. See also: :ref:`always_chunking` and :ref:`a_status_oddity`

The files cache is a python dictionary, storing python objects, which
generates a lot of overhead.

Borg can also work without using the files cache (saves memory if you have a
lot of files or not much RAM free), then all files are assumed to have changed.
This is usually much slower than with files cache.

The **chunks cache** is stored in ``cache/chunks`` and is used to determine
whether we already have a specific chunk, to count references to it and also
for statistics.

The chunks cache is a key -> value mapping and contains:

* key:

  - chunk id_hash
* value:

  - reference count
  - size
  - encrypted/compressed size

The chunks cache is a hashindex, a hash table implemented in C and tuned for
memory efficiency.

The **repository index** is stored in ``repo/index.%d`` and is used to
determine a chunk's location in the repository.

The repo index is a key -> value mapping and contains:

* key:

  - chunk id_hash
* value:

  - segment (that contains the chunk)
  - offset (where the chunk is located in the segment)

The repo index is a hashindex, a hash table implemented in C and tuned for
memory efficiency.


Hints are stored in a file (``repo/hints.%d``).

It contains:

* version
* list of segments
* compact

hints and index can be recreated if damaged or lost using ``check --repair``.

The chunks cache and the repository index are stored as hash tables, with
only one slot per bucket, but that spreads the collisions to the following
buckets. As a consequence the hash is just a start position for a linear
search, and if the element is not in the table the index is linearly crossed
until an empty bucket is found.

When the hash table is filled to 75%, its size is grown. When it's
emptied to 25%, its size is shrinked. So operations on it have a variable
complexity between constant and linear with low factor, and memory overhead
varies between 33% and 300%.

.. _cache-memory-usage:

Indexes / Caches memory usage
-----------------------------

Here is the estimated memory usage of |project_name| - it's complicated:

  chunk_count ~= total_file_size / 2 ^ HASH_MASK_BITS

  repo_index_usage = chunk_count * 40

  chunks_cache_usage = chunk_count * 44

  files_cache_usage = total_file_count * 240 + chunk_count * 80

  mem_usage ~= repo_index_usage + chunks_cache_usage + files_cache_usage
             = chunk_count * 164 + total_file_count * 240

Due to the hashtables, the best/usual/worst cases for memory allocation can
be estimated like that:

  mem_allocation = mem_usage / load_factor  # l_f = 0.25 .. 0.75

  mem_allocation_peak = mem_allocation * (1 + growth_factor)  # g_f = 1.1 .. 2


All units are Bytes.

It is assuming every chunk is referenced exactly once (if you have a lot of
duplicate chunks, you will have less chunks than estimated above).

It is also assuming that typical chunk size is 2^HASH_MASK_BITS (if you have
a lot of files smaller than this statistical medium chunk size, you will have
more chunks than estimated above, because 1 file is at least 1 chunk).

If a remote repository is used the repo index will be allocated on the remote side.

The chunks cache, files cache and the repo index are all implemented as hash
tables. A hash table must have a significant amount of unused entries to be
fast - the so-called load factor gives the used/unused elements ratio.

When a hash table gets full (load factor getting too high), it needs to be
grown (allocate new, bigger hash table, copy all elements over to it, free old
hash table) - this will lead to short-time peaks in memory usage each time this
happens. Usually does not happen for all hashtables at the same time, though.
For small hash tables, we start with a growth factor of 2, which comes down to
~1.1x for big hash tables.

E.g. backing up a total count of 1 Mi (IEC binary prefix i.e. 2^20) files with a total size of 1TiB.

a) with ``create --chunker-params 10,23,16,4095`` (custom, like borg < 1.0 or attic):

  mem_usage  =  2.8GiB

b) with ``create --chunker-params 19,23,21,4095`` (default):

  mem_usage  =  0.31GiB

.. note:: There is also the ``--no-files-cache`` option to switch off the files cache.
   You'll save some memory, but it will need to read / chunk all the files as
   it can not skip unmodified files then.

Encryption
----------

.. seealso:: The :ref:`borgcrypto` section for an in-depth review.

AES_-256 is used in CTR mode (so no need for padding). A 64 bit initialization
vector is used, a MAC is computed on the encrypted chunk
and both are stored in the chunk. Encryption and MAC use two different keys.
Each chunk consists of ``TYPE(1)`` + ``MAC(32)`` + ``NONCE(8)`` + ``CIPHERTEXT``:

.. figure:: encryption.png

In AES-CTR mode you can think of the IV as the start value for the counter.
The counter itself is incremented by one after each 16 byte block.
The IV/counter is not required to be random but it must NEVER be reused.
So to accomplish this |project_name| initializes the encryption counter to be
higher than any previously used counter value before encrypting new data.

To reduce payload size, only 8 bytes of the 16 bytes nonce is saved in the
payload, the first 8 bytes are always zeros. This does not affect security but
limits the maximum repository capacity to only 295 exabytes (2**64 * 16 bytes).

Encryption keys (and other secrets) are kept either in a key file on the client
('keyfile' mode) or in the repository config on the server ('repokey' mode).
In both cases, the secrets are generated from random and then encrypted by a
key derived from your passphrase (this happens on the client before the key
is stored into the keyfile or as repokey).

The passphrase is passed through the ``BORG_PASSPHRASE`` environment variable
or prompted for interactive usage.

.. _key_files:

Key files
---------

.. seealso:: The :ref:`key_encryption` section for an in-depth review of the key encryption.

When initialized with the ``init -e keyfile`` command, |project_name|
needs an associated file in ``$HOME/.config/borg/keys`` to read and write
the repository. The format is based on msgpack_, base64 encoding and
PBKDF2_ SHA256 hashing, which is then encoded again in a msgpack_.

The same data structure is also used in the "repokey" modes, which store
it in the repository in the configuration file.

The internal data structure is as follows:

version
  currently always an integer, 1

repository_id
  the ``id`` field in the ``config`` ``INI`` file of the repository.

enc_key
  the key used to encrypt data with AES (256 bits)

enc_hmac_key
  the key used to HMAC the encrypted data (256 bits)

id_key
  the key used to HMAC the plaintext chunk data to compute the chunk's id

chunk_seed
  the seed for the buzhash chunking table (signed 32 bit integer)

These fields are packed using msgpack_. The utf-8 encoded passphrase
is processed with PBKDF2_ (SHA256_, 100000 iterations, random 256 bit salt)
to derive a 256 bit key encryption key (KEK).

A `HMAC-SHA256`_ checksum of the packed fields is generated with the KEK,
then the KEK is also used to encrypt the same packed fields using AES-CTR.

The result is stored in a another msgpack_ formatted as follows:

version
  currently always an integer, 1

salt
  random 256 bits salt used to process the passphrase

iterations
  number of iterations used to process the passphrase (currently 100000)

algorithm
  the hashing algorithm used to process the passphrase and do the HMAC
  checksum (currently the string ``sha256``)

hash
  HMAC-SHA256 of the *plaintext* of the packed fields.

data
  The encrypted, packed fields.

The resulting msgpack_ is then encoded using base64 and written to the
key file, wrapped using the standard ``textwrap`` module with a header.
The header is a single line with a MAGIC string, a space and a hexadecimal
representation of the repository id.

Compression
-----------

|project_name| supports the following compression methods:

- none (no compression, pass through data 1:1)
- lz4 (low compression, but super fast)
- zlib (level 0-9, level 0 is no compression [but still adding zlib overhead],
  level 1 is low, level 9 is high compression)
- lzma (level 0-9, level 0 is low, level 9 is high compression).

Speed:  none > lz4 > zlib > lzma
Compression: lzma > zlib > lz4 > none

Be careful, higher zlib and especially lzma compression levels might take a
lot of resources (CPU and memory).

The overall speed of course also depends on the speed of your target storage.
If that is slow, using a higher compression level might yield better overall
performance. You need to experiment a bit. Maybe just watch your CPU load, if
that is relatively low, increase compression until 1 core is 70-100% loaded.

Even if your target storage is rather fast, you might see interesting effects:
while doing no compression at all (none) is a operation that takes no time, it
likely will need to store more data to the storage compared to using lz4.
The time needed to transfer and store the additional data might be much more
than if you had used lz4 (which is super fast, but still might compress your
data about 2:1). This is assuming your data is compressible (if you backup
already compressed data, trying to compress them at backup time is usually
pointless).

Compression is applied after deduplication, thus using different compression
methods in one repo does not influence deduplication.

See ``borg create --help`` about how to specify the compression level and its default.

Lock files
----------

|project_name| uses locks to get (exclusive or shared) access to the cache and
the repository.

The locking system is based on creating a directory `lock.exclusive` (for
exclusive locks). Inside the lock directory, there is a file indicating
hostname, process id and thread id of the lock holder.

There is also a json file `lock.roster` that keeps a directory of all shared
and exclusive lockers.

If the process can create the `lock.exclusive` directory for a resource, it has
the lock for it. If creation fails (because the directory has already been
created by some other process), lock acquisition fails.

The cache lock is usually in `~/.cache/borg/REPOID/lock.*`.
The repository lock is in `repository/lock.*`.

In case you run into troubles with the locks, you can use the ``borg break-lock``
command after you first have made sure that no |project_name| process is
running on any machine that accesses this resource. Be very careful, the cache
or repository might get damaged if multiple processes use it at the same time.
