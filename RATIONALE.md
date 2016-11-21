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

Failures of `sh`
----------------

The standard shell as specified by the POSIX has multiple issues that can't be
addressed without a complete rewrite.

### Its syntax is cryptic

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

    $a=\a            # '=a: command not found'
    a=\a             # Doesn't work, saves string 'a' instead.
    a="\a"           # Doesn't work, saves literal string '\a'.
    a=`printf \a`    # Doesn't work, saves 'a' again.
    a=`printf "\a"`  # Works.
    a =`printf "\a"` # Doesn't work (runs command `a`)
    a= `printf "\a"` # Doesn't work (runs command `\a` with `a` set to "")
    a=`printf \\\a`  # Also works.
    a=$(printf \\a)  # Works, too.
    a=(printf \\a)   # Unspecified.

Even with this tiny example, it is difficult to predict the behaviour of the
program. And the syntax elements demonstrated by these commands isn't even an
insignificant one: all the presented syntax elements occur frequently in
scripts on quite a larger scale.

There are many constructs with little semantics to distinguish between them.
Understanding how the command strings are evaluated certainly helps, but even
then there are some corner cases that are hard to remember.

### Redirection is uncomfortable

#### Pipes with source other than the standard output

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

#### Connecting two commands at both ends

Let's say we have a program `rename` which accepts a rule for how to transform
filenames and a list of files to rename according to that rule. We'll describe
multiple possible solutions later, in discussion of a general lack of
first-class functions, but there is a particular one that's worth mentioning
here.

`rename` can write to stream `3` and read the result from stream `4`. The
solution is nice and general: in theory, we can plug any command at both these
ends, including pipes in subshells.

Now, how _do_ we redirect?

One solution we've found is to use named pipes:

    mkfifo o3 i4
    transform < o3 > i4 & rename 3> o3 4< i4 
    rm o3 i4

One can do with a single pipe, too, simultaneously avoiding putting the process
in the background, but at the cost of rendering the command unreadable:

    mkfifo i4
    (rename 1>&5 3>&1 4< i4 | transform > i4) 5>&1
    rm i4

If `rename` is a shell script, implementing its relationships with the third
and fourth streams is also not an easy task. A script which simply pipes its
argument if it's placed in this system instead of `rename` may look like this:

    printf "%s" "$1" 1>&3
    exec 3>&-
    cat <&4

If the `exec` is omitted, `transform` can await the input which never comes if
it doesn't close its `stdin` first. Most filters don't close their `stdin`
during normal operation.

These abstraction are quite low-level, and even simple tasks which don't have
some syntactic support will lead to code that looks like Cthulhu.

### Code abstraction is bothersome

`sh` requires one to perform many operations by hand, and so some mechanisms of
separating the concepts that are used often would be helpful. And there are
some, but using them is more often than not is harder than typing everything by
hand each time.

Let's say we are concerned with privacy and want to encrypt and decrypt some
sensitive files on a regular basis with these commands:

    gpg -aco "$file".gpg "$file" && shred -uz "$file"
    gpg  -do "$file" "$file".gpg && shred -uz "$file".gpg

It seems natural to avoid typing all this every time and instead have something
like this:

    enc "$file"
    dec "$file".gpg

#### Aliases are just string substitutions

Aliases aren't sufficient for this task. The best we can do is

    alias enc="gpg -ac"
    alias dec="gpg -d"

The reason is that aliases don't accept arguments. The best they can do is
substitute some string at the beginning of a command with another string. They
have their uses:

    alias ls="ls --color --classify -C"

For anything even slightly more complex, one has to define a function.

#### Functions require manual checks

Functions in `sh` are really plain. They accept a list of arguments, initialize
structures for access to them, and execute a sequence of commands.

Let's try to use the naive approach:

    enc() { gpg -aco "$1".gpg "$1"    && shred -uz "$1"; }
    dec() { gpg -do  "${1%.gpg}" "$1" && shred -uz "$1"; }

These functions make several dangerous assumptions. For example, if an
encrypted file doesn't have the `.gpg` extension, `dec` removes it altogether.
And if no arguments are passed into the functions, the errors aren't really
helpful for someone who doesn't know what's going on. Or if you pass these
functions a symlink, the files to which they point are overwritten but not
removed. If two or more arguments are passed into these functions, only the
first one is considered, the rest are silently ignored.

One can write something like this:

    enc() {
        if [ "$#" -ne 1 ]; then
            printf "%s\n" "Expected exactly one argument"
            return 1
        elif [ ! -f "$1" ]; then
            if [ -e "$1" ]; then
                printf "%s\n" "$1 doesn't exist"
            else
                printf "%s\n" "$1 is not a plain file"
            fi
            return 1
        fi 1>&2
        gpg --armor --symmetric --output "$1".gpg "$1" && shred -uz "$1"
    }
    dec() {
        if [ "$#" -ne 1 ]; then
            printf "%s\n" "Expected exactly one argument"
            return 1
        elif [ ! -f "$1" -o -L "$1" ]; then
            if [ -e "$1" ]; then
                printf "%s\n" "$1 doesn't exist"
            else
                printf "%s\n" "$1 is not a plain file"
            fi
            return 1
        elif [ "${1%.gpg}".gpg != "$1" ]; then
            printf "%s\n" "$1 doesn't have the extension .gpg"
            return 1
        fi 1>&2
        gpg --decrypt --output "${1%.gpg}" "$1" && shred -uz "$1"
    }

Some parts of it can be slightly improved, but the overall mass of this
structure is overwhelming. Oftentimes the cost of supporting a number of long
functions outweighs the benefits of code abstraction, forcing users to stick to
only the simplest substitutions in most cases.

Of course, there always are some possibilities for abstraction. For example,
one could write

    _is_1_and_plain() {
        if [ "$#" -ne 1 ]; then
            printf "%s\n" "Expected exactly one argument"
            return 1
        elif [ ! -f "$1" -o -L "$1" ]; then
            if [ -e "$1" ]; then
                printf "%s\n" "$1 doesn't exist"
            else
                printf "%s\n" "$1 is not a plain file"
            fi
            return 1
        fi 1>&2
    }
    _has_extension() {
        if [ "${1%.$2}".$2 != "$1" ]; then
            printf "%s\n" "$1 doesn't have the extension .$2"
            return 1
        fi 1>&2
    }
    enc() {
        _is_1_and_plain "$@" &&
            gpg --armor --symmetric --output "$1".gpg "$1" && shred -uz "$1"
    }
    dec() {
        _is_1_and_plain "$@" && _has_extension "$1" gpg &&
            gpg --decrypt --output "${1%.gpg}" "$1" && shred -uz "$1"
    }

The problem is that it's unlikely that many functions shall require a single
argument with a particular extension, and so the benefits of abstracting the
code aren't substantial.

#### First-class function support is really bad

First-class functions are a really natural concept of abstracting common data
manipulation paradigms. The core idea is telling one function which other
function to call. For example, `find -exec` and `xargs` both accept as
parameters the names of programs to be run, and so they can be considered
higher-order functions.

But what about passing compound commands to a program? This can be done, too,
by first creating a file containing the commands, and then passing that.

    echo 'grep TODO "$1" | wc -l' > count_todo.sh
    chmod +x count_todo.sh
    find . -name "*.sh" -print -exec "$(pwd)/count_todo.sh" '{}' ';'

In the discussion of bad piping facilities we've mentioned that one can create
higher-order functions with pipes, but this only works feasibly in cases when
one invocation of the function being passed is enough. In this example, the
whole pipe must be run anew for every discovered file. We don't know of any
general solution for this problem.

Of course, one could also call `sh -c` which solves most of the problems shell
users can have.

    find . -name "*.sh" -print -exec sh -c "grep TODO '{}' | wc -l" ';'

The main problem with this approach is having another layer of expansions and
substitutions. Where will this variable expand, in this shell or in the inner
one? How much backslashes must one put to have a literal backslash in the `sed`
expression written between single quotes in the string wrapped in double quotes
and used as argument to `sh -c` which is itself inside backticks? Not an easy
question but, unluckily, one that isn't of pure academic interest and arises in
day-to-day use. Usually the answer is trying everything on simple examples and
praying that larger ones won't cause some hidden aspect of complexity to
emerge. We've observed that the amount of time needed to debug a shell command
is in exponential relation with the number of layers in it, and each `sh -c` or
`eval` opens up a great number of them.

It should be noted that the difficulty of passing functions to programs must
be primarily attributed to the lack of corresponding call conventions. Shell
shouldn't invent its own form of passing arguments to programs. But even where
the shell itself is concerned, the situation isn't much more bright.

It is true that one is able to avoid writing compound expressions to a file,
defining a function instead:

    count_todo() { grep TODO "$1" | wc -l; }
    ls_with_op() {
        for i in `ls`; do "$@" "$i"; done
        # don't do "for i in `ls`" at home!
    }
    ls_with_op echo
    ls_with_op count_todo
    ls_with_op mv -t ~
    ls_with_op { echo "$1" } # Fails

But one can't pass anonymous functions. There is no option to simply write

    ls_with_op (mv -vi "$1" $(printf "%s" "$1" | sed "s/^0//g"))

The transformation described here is simple and common: renaming multiple
files at the same time according to some rule, here it removing leading zero
from the name.

There are many ways to express this concept. One possible workaround is

    ls -1 | sed 's/^0\(.*\)/0\1\x0\1\x0/' | tr -d '\n' | xargs -0 -n2 mv

And `xargs` often does save the day, but not often enough. And look at this
line!

### Environment is implicitly passed

There are many settings which it's reasonable to define globally. For example,
passing the language in which error messages should be displayed is extremely
cumbersome, and each and every program that calls other programs would have to
remember to pass this option everywhere. And it's bad enough with one option,
but they are numerous!

  * Language of the messages;
  * List of directories in which to look for executable files (in order to be
    able to type `cat` instead of `/bin/cat`);
  * Address of the X11 server;
  * Type of the terminal;
  * Name of the default editor and Internet browser;
  * Many-many others.

Storing these settings in per-user configuration files isn't viable: often we
need to be able to modify these variables for a specific program at a specific
point of time. For example, one could see error messages in one language, and
then, failing to identify the problem, switch to English in order to search the
Internet for a solution. Another example is scripts that runs programs which
aren't really useful elsewhere, so users don't want these programs to appear in
their auto-completion but it's reasonable for script to be able to address the
programs by the full names.

The present solution for this problem is environment. An environment of a
process is a set of key-value pairs which can be modified by the process and
is inherited by the processes called by it. So we may just call

    LANG=en_US.UTF8 sh

and all the programs that are called from this shells shall be able to query
the value of `$LANG` and, if they support this variable, output their messages
in English.

However, there is a major problem associated with this. Environment is just
*too* transparent. It adds a new level of indirection in program runtime but
isn't explicitly registered anywhere. If a session was closed, there is no way
to know with which environment has that line of `sh` history been executed, and
way too often users don't bother with checking the output of `env` after
encountering a bug in a program.

The result is a plethora of bugs that occur only when a specific set of
environment variables has specific values because, contrary to testing with
wrong values in program arguments or standard input, testing with wrong
environment variables is just too hard: one has to remember that the
environment is inherited by the child programs and which environment settings
have which effect in them. Command-line parameters, on the other hand, are
explicitly set by the programmer while implementing the calling behaviour,
hence it's impossible to disregard them.

It's true that when software doesn't work properly a specific, magical, set of
environment variables may cause it to produce the desired results. But it is
the case of flexibility at the expense of robustness: the astonishing amount of
hidden behaviour which surfaces once you've set some environment variables
makes determining the cause and effect quite painful.

