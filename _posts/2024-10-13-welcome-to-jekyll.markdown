---
layout: post
title:  "What's a repeatable read?"
date:   2024-10-13 15:06:35 -0400
categories: database isolation repeatable read
---

Hello world! I am kicking off my blog as part of an online bookclub that
I just joined. Phil Eaton is hosting the book club, and this go around
we are covering "Database Design and Implementation" by Edward Sciore.

Chapter 2 of the book covers isolation levels, and in particular it
mentioned the notion of repeatable reads. I happened to be reading several
other sources, and at first blush it seems to me like there isn't a
definite consensus on what a repeatable read really is. In fact, one of
the other books I'm reading, "Designing Data Intensive Applications"
by Martin Kleppmann, goes so far as to say that "nobody really knows what
a repeatable read means." So this is basically my starting hypothesis,
that there probably isn't a standard definition for what it is. But I'd like
to arrive to that conclusion myself to see how that came to be.

This post assumes some background on transaction serializability and
isolation levels, but as a quick summary, the idea is this: a database
can execute multiple transactions concurrently, and if we allow arbitrary
interleaving of concurrently executing transactions, problems can pop up.
The strictest isolation level is called serializable. What this means is
if we concurrently execute transactions, the result should be the same as
if we executed the transactions in some order, one after the other, without
any concurrency. More formally, we say that a schedule we allow to execute
is conflict-equivalent to a serial schedule. More information can be found
in chapter 17 of the Database System Concepts textbook.

Here is a simple example, suppose we have two transactions accessing a
database item X.

Transaction 1: ```<START1> <WRITE1 X> <READ1 X> <COMMIT1>```<br>
Transaction 2: ```<START2> <WRITE2 X> <COMMIT2>```<br>

And here is what a concurrent interleaving may look like (abbreviated):

Schedule 1: ```<S1> <S2> <W1 X> <W2 X> <C2> <R1 X> <C1>```

There's a problem here, namely in between transaction 1 writing the
initial value of x and then reading it again, transaction 2 jumps in the
middle and writes its own value of X. In a serial schedule, we would either
want transaction 2 to write the value of X either before transaction 1
first writes it, or after transaction 1 reads its own write.

As I was surprised to learn, however, this fully serializable scheduling
isn't necessarily the default in many databases. It turns out we can relax
the serializability restriction and allow some conflicts to occur, basically
putting the onus upon the application developer to know what level of
isolation they are working with, and make sure bad stuff doesn't happen.

In the sequel, I want to explore how the repeatable read level is defined,
and see what the similarities or differences are between definitions and
implementations that vendors provide.

First I'll start with chapter 2 of Sciore's book, where he defined ```repeatable
reads``` as follows: ```it extends the read-commited isolation level such that
reads are always "repeatable". In turn, read-commited isolation is defined
as forbidding a transaction from accessing uncommited values.```

If you return to schedule 1 from above:

Schedule 1: ```<S1> <S2> <W1 X> <W2 X> <C2> <R1 X> <C1>```

This is actually okay under the read-commited isolation level. Since
transaction 2 made a commit of its write, transaction 1 is allowed to read
the new value of X even though it makes its initial write ```<W1 X>```
pointless for the transaction. Is this schedule also allowed with
repeatable reads?

Now the definition provided is perhaps a
bit vague, since it simply states that reads are repeatable.
To clarify the intended definition, I will provide the definition of
repeatable reads from chapter 17 of Database System Concepts. None of these
working definitions are meant to be the definitive version of what a
repeatable read is, I am simply taking different definitions and exploring
the consequences of each. This one is peculiar because it is simple to
understand, but interestingly doesn't show up anywhere else:

**Working Definition 1:**

```Repeatable read allows only commited data to be read, with the further
stipulation that between two reads of a data item by the same transaction,
no other transaction is allowed to update it.```

This provides a little more clarity on what is meant by a read being
'repeatable'. If you check schedule 1 again, we aren't doing more than
one read, so the schedule is also okay under repeatable read. Let's modify
the transactions a bit to explore repeatable reads:

Transaction 1: ```<START1> <WRITE1 X> <READ1 X> <READ1 X> <COMMIT1>```<br>
Transaction 2: ```<START2> <WRITE2 X> <COMMIT2>```<br>

Here we just inserted a second read operation for the first transaction. Now if we were to have the following schedule:

Schedule 1: ```<S1> <S2> <W1 X> <R1 X> <W2 X> <C2> <R1 X> <C1>```

we have a schedule that is not permissible under repeatable reads, but would
be allowed under a read-commited isolation level. After the first read
for transaction 1, transaction 2 writes the value of X thus breaking the
repeatable read constraint. At first glance, this isolation level may seem
pretty close to full serializability, but we can show an example that
breaks serializability:

Schedule 2: ```<S1> <S2> <W1 X> <W2 X> <C2> <R1 X> <R1 X> <C1>```

Here transaction 1 writes the value for X, and then transaction 2 jumps in
to write its own value and commits. Since transaction 2 performed a commit
operation, and it happened before the first read for transaction 1,
the repeatable read isolation level is satisfied here. Repeatable read
only says that once a transaction starts reading a location X, we don't want
other transactions to overwrite it between successive reads.

This definition seems okay, and easy to understand. However things start
to get a little confusing when you start looking at even more definitions.
In fact, I'm not sure if anyone uses the definition I introduced from
the database system concepts textbook. It was a a starting point for some
intuition, but let's keep exploring.

Let's start with how [PostgresSQL](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-REPEATABLE-READ) defined repeatable reads:

**Working Definition 2:**

```The repeatable read isolation level only sees data commited before the
transaction began; it never sees either uncommited data or changes
committed by concurrent transactions during the transaction's execution.```

This is different from the first working definition, because now we cannot
allow another transaction to make updates before the first read. Any
conflicting updates have to happen before the transaction starts. Let's
modify schedule 2 now to make it work under this definition of repeatable
reads:

Schedule 3: ```<S2> <W2 X> <C2> <S1> <W1 X> <R1 X> <R1 X> <C1>```

Now we are okay; transaction 2 wrote to X and commited before transaction
1 even began. However we can also observe something else: this schedule
is serial! So what are the potential issues that can pop up under this
definition that would cause conflicts? 





