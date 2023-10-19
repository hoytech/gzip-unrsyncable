# gzip --UNrsyncable

This is a proof-of-concept perl script that manipulates a data stream in order to prevent [gzip --rsyncable](https://beeznest.wordpress.com/2005/02/03/rsyncable-gzip/) from working properly.

Consider a sequence that consists of 100,000 `A` characters. Needless to say, `gzip` will compress this extremely well:

    $ perl -e 'print "A"x100_000' | gzip -c | wc -c
    133

`gzip` provides a special `--rsyncable` option that will flush the compressed stream periodically, so that compressed blocks of data are preserved and can be found by `rsync`'s rolling checksum. This adds a small amount of overhead to the compressed stream:

    $ perl -e 'print "A"x100_000' | gzip -c --rsyncable | wc -c
    3020

The way `gzip --rsyncable` works is by looking for particular patterns in its input data. Specifically, it queus a stream flush whenever the last 4096 bytes sum to 0 (mod 4096).

The `gzip-unrsyncable` script also looks for these patterns and, when it detects one, modifies the last byte to break the pattern. This prevents `gzip --rsyncable` from flushing its stream:

    $ perl -e 'print "A"x100_000' | ./gzip-unrsyncable | gzip -c --rsyncable | wc -c
    192

Mark Adler's [infgen](https://github.com/madler/infgen) utility can be used to verify this. Normally the stream would be flushed 746 times (the last `end` line is the actual end):

    $ perl -e 'print "A"x100_000' | gzip -c --rsyncable | infgen | grep ^end | wc -l
    747

However, with `gzip-unrsyncable`, the stream is never flushed:

    $ perl -e 'print "A"x100_000' | ./gzip-unrsyncable | gzip -c --rsyncable | infgen | grep ^end | wc -l
    1

Note that there are also "naturally occurring" unrsyncable sequences, for example:

    $ perl -E 'print chr($_ % 256) for (1..100_000)' | gzip -c --rsyncable | infgen | grep ^end | wc -l
    1
