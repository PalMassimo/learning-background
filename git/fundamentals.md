# Fundamentals

## Hash Algorithms
Most common hash algorithms are

- `MD5` -> 128 bits
- `SHA1` -> 160 bits
- `SHA256` -> 256 bits
- `SHA512` -> 512 bits

Git uses `sha1` to compute the file hashes. In hexadecimal notation, sha1 hashed are 40 long.

## Git Object Types
Git has four object types:
- blob: files
- tree: git inner filesystem
- commit
- annotated tags

## Basic commands

We use `git hash-object` command to generate (and store with `-w` option) a blob object with its corresponding hash

```bash
$ git hash-object --stdin -w # generates hash from stdin
$ git hash-object <file-name> -w 
```

To get information about blob objects run `git cat-file` command

```bash
$ git cat-file -p <hash> # print content
$ git cat-file -s <hash> # print size 
$ git cat-file -t <hash> # print type object
