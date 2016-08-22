Rationale for flush
===================

The evolution of madness
------------------------

We firmly believe that Haskell is more comfortable than C in terms of
simplicity of syntax, expressiveness, degree to which the compiler eliminates
possible errors, helpfulness of possible optimisations, and overall ease of
use.

This isn't a failure of C itself, but rather the natural consequence of the
fact that C has had its highest points at times when hardware was constantly
and significantly evolving, forcing the language to evolve as well, often
without it being ready for such changes.

The shell in general as we know it today is also a product of such an
evolution. The Thompson shell was supplemented by the Bourne shell, which was
followed the POSIX shell standard to which adhere Bourne-again shell, Z shell,
and many others. Many elements of the modern shells have the same semantics and
syntax as the first shell released in 1971.

The result as we have it is a big mess of features which had been continuously
thrown on top of the old ones, often without possibility to make these changes
consistent with existing language. Is it a failure of the Thompson shell? Not
at all, it simply couldn't foresee the changes that would come later, and
sometimes the effort to predict them would hinder the immediate creation of the
revolutionary features it introduced.

We believe that there exists the Haskell of shells, a language with few core
concepts but much freedom in combining them. Our goal is to find, document, and
implement it.

Failures of existing shells
---------------------------

### `sh`

The standard shell as specified by the POSIX has multiple issues that can't be
addressed without a complete rewrite.

#### Its syntax is cryptic

Here are three different commands:

    echo $a
    echo "$a"
    echo '$a'

Anyone with basic familiarity with shell scripting would immediately
distinguish them.

Here are another three different commands:

    for a in "$*"; do echo "$a"; done
    for a in "$@"; do echo "$a"; done
    for a in  $@ ; do echo "$a"; done

Specifying the difference between them is a little harder.

And what do these do?

    a=${v:-3}
    b=${v:+3}
    c=${PATH##*:}
    d=${PATH%%:*}
    e=${PATH%#:*}

Quite cryptic even though learning these is easy.

What's much worse is syntax that seems very similar but has different meaning.
For this one, let's look at a real task: how does one save the visual bell
character in a variable?

    $a=\a           # '=a: command not found'
    a=\a            # Doesn't work, saves string 'a' instead.
    a="\a"          # Doesn't work, saves literal string '\a'.
    a=`printf \a`   # Doesn't work, saves 'a' again.
    a=`printf "\a"` # Works.
    a=`printf \\\a` # Also works.
    a=$(printf \\a) # Works, too.
    a=(printf \\a)  # Unspecified.

Even with this tiny example, it is difficult to predict the behaviour of the
program. And the syntax elements demonstrated by these commands isn't even an
insignificant one: all the presented syntax elements occur frequently in
scripts on quite a larger scale.

There are many constructs with little semantics to distinguish between them.
Understanding how the command strings are evaluated certainly helps, but even
then there are some corner cases that are hard to remember.

#### Redirection is uncomfortable

##### Pipes with source other than the standard output

One can't pipe the standard error stream to a program. Pipes only work on
standard output.

`pv` is a nice program which prints on the `stderr` the speed with which data
flows through the pipe. Let's say we have a program `notify_on_stall` that
reads its standard input for the output of `pv` and signals the user somehow if
there is no activity for 10 or so seconds. Now we need to run `do_smth` with
the contents of `/dev/input/mice` being directed at its standard input,
signaling the user if `/dev/input/mice` suddenly can't be read.

    (pv -r < /dev/input/mice | do_smth) 2>&1 notify_on_stall

This works. That is, until `do_smth` decides to write something on `stderr`.
Then we need to do something like this:

    (pv -r < /dev/input/mice 2>&3 | do_smth) 3>&1 notify_on_stall

But then let's assume that it has to write something on the standard output! We
have to resort to this:

    ((pv -r < /dev/input/mice 2>&3 | do_smth 1>&4) 3>&1 notify_on_stall) 4>&1

The whole expression can probably qualify for “cryptic syntax” (see above). It
is uncommon to meet a man who instantaneously grasps what this code does.

##### Connecting two commands at both ends

Let's say we have a program `rename` which accepts a rule for how to transform
filenames and a list of files to rename according to that rule. We'll describe
multiple possible solutions later, in discussion of a general lack of
higher-order functions, but there is a particular one that's worth mentioning
here.

`rename` can write to stream `3` and read the result from stream `4`. The
solution is nice and general: in theory, we can plug any command at both these
ends, including pipes in subshells.

Now, how _do_ we redirect?

The only solution we've found is to use named pipes:

    mkfifo o3 i4
    transform < o3 > i4 & rename 3> o3 4< i4 
    rm o3 i4

If `rename` is a shell script, implementing its relationships with the third
and fourth streams is also not an easy task. A script which simply pipes its
argument if it's placed in this system instead of `rename` may look like this:

    printf "%s" "$1" 1>&3 & exec 3>&-; cat <&4

If the `exec` is omitted, `transform` can await the input which never comes if
it doesn't close its `stdin` first. Most filters don't close their `stdin`
during normal operation.

These abstraction are quite low-level, and even simple tasks which don't have
some syntactic support will lead to code that looks like Cthulhu.

