Functional, Lazy, and Usable SHell (flush)
==========================================

This is a repository for a brand new shell called `flush`.

Design goals
------------

  * Simple and transparent syntax
  * Strong typing with ad hoc polymorphism
  * First-class functions, upport of pattern matching, lazy evaluation, and
    other concepts typical of functional languages
  * First-class environments

State of the project
--------------------

Currently there is no realisation of the shell, and none is planned in the near
future. The reason for this is that the shell is an environment in which users
spend much of their time, and even the smallest usability problems can become a
major nuisance. Much thought is required in order to avoid creating the
JavaScript of shells, so all the available code is highly experimental and not
meant for everyday use. The questions of performance are not raised at all,
and, for example, taking a minute to parse the input is not a bug (even though
this particular anomaly shall supposedly never occur).

See also
--------

  * [The rationale](RATIONALE.md)

