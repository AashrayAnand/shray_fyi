+++
title = "My (Mostly Distributed Systems) Reading List"
date = 2025-07-01
+++

I have had tons of different bookmarks in my reading list to eventually get back to and read but they get mixed up and shifted around and I naturally let my ADHD-infused thought process clutter it up and randomize my time. I am going to try and avoid that by dumping my reading list into the blog directly, and forcing myself to get through everything here in the order that I have laid out from the beginning, rather than constantly randomizing over the endless different things I eventually want to learn.

Distributed systems is the primary driver of this reading list, but there are other tidbits and tangents along the way that I want to get back to that are also captured here.

1. [Raft Paper](https://raft.github.io/raft.pdf) -> Currently re-reading this, this time I am implementing it from hand as well.
2. [Paxos in Paper Trail](https://www.the-paper-trail.org/post/2009-02-03-consensus-protocols-paxos/) -> Re-reading this as a follow up to the original Paxos paper, and before implementing this by hand. 
3. [Chubby Lock Manager](https://storage.googleapis.com/gweb-research2023-media/pubtools/4444.pdf)
4. [Attention is All you Need](https://proceedings.neurips.cc/paper_files/paper/2017/file/3f5ee243547dee91fbd053c1c4a845aa-Paper.pdf) -> Trying to make more time for AI-related theory.
5. [interpreter book](https://edu.anarcho-copy.org/Programming%20Languages/Go/writing%20an%20INTERPRETER%20in%20go.pdf) and [compiler book](https://compilerbook.com/) -> Some weekend reading when I get a chance to hack these together, preferably in Rust.

Once the list above is exhausted will I revisit this list and add some more new topics from:

1. [Awesome DS](https://github.com/theanalyst/awesome-distributed-systems)
2. [Paper Trail DS Theory](https://www.the-paper-trail.org/post/2014-08-09-distributed-systems-theory-for-the-distributed-systems-engineer/)
3. [Murat's Foundational DS List](https://muratbuffalo.blogspot.com/2021/02/foundational-distributed-systems-papers.html)
4. [A DS Reading List](https://dancres.github.io/Pages/)
5. [Reynold Xin Database Papers](https://github.com/rxin/db-readings)
6. [Mark Brooker's Blog](https://brooker.co.za/blog/)
7. [Christopher Meiklejohn DS Readings](https://christophermeiklejohn.com/distributed/systems/2013/07/12/readings-in-distributed-systems.html)

I also want to use this list to archive some of the various things I have read before and may eventually want to revisit:

1. [Designing Data-Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/) -> The primer book that I have learned most DS theory from, and what helps set me up to go further on any specific paper or implementation.
2. [What every Programmer Should Know about Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) -> Ulrich Drepper's dictionary on the fundamentals of memory that every programmer should know. Super dense but full of tons of super useful things to revisit.
3. [CAP in Plain English](http://ksat.me/a-plain-english-introduction-to-cap-theorem) -> The write up on CAP that solidified it a lot for me, along with the DDIA description, roughly "In the face of P, we can strictly choose C or A."
4. [Jepsen's DS Class](https://github.com/aphyr/distsys-class) -> Jepsen's dictionary on distributed systems, good to revisit various terminology and the basics.
5. [CHORD](https://pdos.csail.mit.edu/papers/chord:sigcomm01/chord_sigcomm.pdf) -> Pretty cool paper and sets the stage for what Dynamo talks about.
6. [Dynamo](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) -> Not DynamoDB! This and CHORD were some pretty interesting first papers I read and implemented.
7. [Google File System](https://static.googleusercontent.com/media/research.google.com/en/us/archive/gfs-sosp2003.pdf) -> GFS's general design for distributed append-only storage chunking and relaxed consistency is a pattern I feel like I see all over the place elsewhere.
8. [Spanner](https://static.googleusercontent.com/media/research.google.com/en/us/archive/spanner-osdi2012.pdf) -> The idea of planet-scale transactions and the ability to assign accurate timestamps is what makes this paper pretty awesome.
9. [FLP Paper](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf) -> More of a core idea to remember than a paper I constantly revisit. Async systems can't guarantee consensus if any process can fail. [Paper Trail's Summary of FLP](https://www.the-paper-trail.org/post/2008-08-13-a-brief-tour-of-flp-impossibility/) is also good.
10. [Snowflake](https://www.cs.cmu.edu/~15721-f24/papers/Snowflake.pdf) -> How Snowflake (similarly to Aurora) created a new mode of operation for cloud databases, this time elastically scaling data warehousing.
11. [Aurora](https://pages.cs.wisc.edu/~yxy/cs764-f20/papers/aurora-sigmod-17.pdf) -> The database I actively work on, and an extremely innovative idea that changed how databases operate in the cloud. In short, why not let storage be able to replay the log, then we can separate storage and compute as distinctly elastic resources. You know it's genius because everyone has copied it since.
12. [High Performance Browser Networking](https://hpbn.co/)
13. [Cassandra](https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf) -> How I learned about LSM trees and first started understanding storage system architectures better.
14. [Bitcask](https://riak.com/assets/bitcask-intro.pdf) -> Simpler than LSM trees, another log-based append storage engine.
15. [ZooKeeper](https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf) -> ZAB i Another classic consensus system that is an interesting highlight of this read.
16. [Lakehouse](https://15721.courses.cs.cmu.edu/spring2024/papers/01-modern/armbrust-cidr21.pdf) -> More in the DB engine and architecture vein. an interesting read to understand where data systems are going in the current day, how can we get it all in terms of data warehousing, ACID, decoupled storage and compute.
17. [MapReduce](https://storage.googleapis.com/gweb-research2023-media/pubtools/4449.pdf) -> Probably the first DS or parallel computing paper I ever read.
