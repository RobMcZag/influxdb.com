+++
title = "The new InfluxDB storage engine: a Time Structured Merge Tree"
author = "Paul Dix"
date = "2015-10-07"
publishdate = "2015-10-07"
+++

For more than a year we've been talking about potentially making a storage engine purpose-built for our use case of time series data. Today I'm excited to announce that we have the first version of our new storage engine available in a nightly build for testing. We're calling it the Time Structured Merge Tree or TSM Tree for short.

In our early testing we've seen up to a 45x improvement in disk space usage over 0.9.4 and we've written 10,000,000,000 (10B) data points (divided over 100k unique series written in 5k point batches) at a sustained rate greater than 300,000 per second on an EC2 c3.8xlarge instance. This new engine uses up to 98% less disk space to store the same data as 0.9.4 with no reduction in query performance.

In this post I'll talk a little bit about the new engine and give pointers to more detailed write-ups and instructions on how to get started with testing it out.

### Columnar storage and unlimited fields

The new storage engine is a columnar format, which means that having multiple fields won't negatively affect query performance. For this engine we've also lifted the limitation on the number of fields you can have in a measurement. For instance, you could have MySQL as the thing you're measuring and represent each of the few hundred different metrics that you gather from MySQL as separate fields.

Even though the engine isn't optimized for updates, the new columnar format also means that it's possible to update single fields without touching the other fields for a given data point.

### Compression

We use multiple compression techniques which vary depending on the data type of the field and the precision of the timestamps. Timestamp precision matters because you can represent them down to nanosecond scale. For timestamps we use delta encoding, scaling and compression using simple8b, run-length encoding or falling back to no compression if the deltas are too large. Timestamps in which the deltas are small and regular compress best. For instance, we can get great compression on nanosecond timestamps if they're only 10ns apart each. We'd achieve the same level of compression for second precision timestamps that are 10s apart.

We use the same delta encoding for floats mentioned in <a href="http://www.vldb.org/pvldb/vol8/p1816-teller.pdf" target="_">Facebook's Gorilla paper</a>, bits for booleans, delta encoding for integers, and Snappy compression for strings. We're also considering adding dictionary style compression for strings, which is very efficient if you have repeated strings.

Depending on the shape of your data, the total size for storage including all tag metadata can range from 2 bytes per point on the low end to more for random data. We found that random floats with second level precision in series sampled every 10 seconds take about 3 bytes per point. For reference, Graphite's Whisper storage uses 12 bytes per point. Real world data will probably look a bit better since there are often repeated values or small deltas.

### LSM Tree similarities

The new engine has similarities with LSM Trees (like LevelDB and Cassandra's underlying storage). It has a write ahead log, index files that are read only, and it occasionally performs compactions to combine index files. We're calling it a Time Structured Merge Tree because the index files keep contiguous blocks of time and the compactions merge those blocks into larger blocks of time.

Compression of the data improves as the index files are compacted. Once a shard becomes cold for writes it will be compacted into as few files as possible, which yield the best compression.

### Testing and more resources

We're announcing the new storage engine early because we want to put both the design and the code out for review and testing in the community. It's not for production use at this point. In fact, it's not enabled by default on the nightly builds and you should plan to blow away all your data between nightly build installs. You can <a href="/docs/v0.9/introduction/tsm_installation.html" target="_">follow these instructions</a> for how to enable the new storage engine and get started testing it.

We've also put up a detailed writeup about what storage engines we tried before, why they didn't work for us and all the in depth specifics about the new <a href="/docs/v0.9/concepts/storage_engine.html" target="_">time structured merge tree engine</a>.

We hope that you'll test it out and let us know how it's working. We're doing extensive testing internally on both raw hardware and in various cloud providers. Once we feel it's ready for more serious use we'll make an announcement here. The 0.9.5 release will ship with this new storage engine along with the ability to do hot backups against it. The release date for 0.9.5 will be "when it's ready." That is, after we've done extensive testing and bug fixing against the new engine.

Finally, we shot a short video of me whiteboarding some of the basics around the design of the new storage engine:

<iframe src="https://player.vimeo.com/video/140372527" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

<br />