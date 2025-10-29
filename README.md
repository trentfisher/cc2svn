# cc2svn -- ClearCase to Subversion #

NOTE: I was hoping to work on this tool, but I never got permission from my company to release it and never had the time to write it.  Hopefully my notes below will explain why we need such a tool and why it is so difficult.

## Why A New Tool? ##

All existing ClearCase to Subversion converters suffer from fatal flaws.  The simplest tools simply import static file trees.  The tools that try to convert history do not properly deal with directory versioning, the fatal flaw being that they do not include the "-all" option to "lshistory".  These tools will fail on all but the simplest projects.

## Why is it so hard? ##

This is a collection of notes about the various corner cases when
converting from ClearCase to Subversion.  And there are a lot of them!

The difficulties are entirely on the ClearCase side, so it doesn't matter whether we convert to Subversion, Git, or whatever is in fashion when you read this.

Remember the old saying "good, fast, cheap, pick two".  Given that we
are working with ClearCase, there is pretty much no chance of "fast".
I am focusing on "good".

### Lshistory is backwards

The lshistory command always shows out in reverse chronological order,
and there is no way to change that.  Therefore we have to dump the
entire history to a file and read it backwards.  This will make
startup slow and will take up extra disk space for large and/or
long-lived vobs.

I have tried to think of various ways around it.  If lshistory
provided a way to specify an end date, the history could be fetched in
chunks going forwards in time and then reverse each chunk.  But, no
luck.

## Lshistory -recurse is wrong

This is the fatal flaw in all other ClearCase to Subversion
converters.  The -recurse option will only show history for elements
visible in the current view.  So elements which have been removed or
elements which only exist on side branches will be entirely
unrepresented.

We have to use the -all option, which will get history on all elements
regardless of visibility.  Elements not visible will have extra
version qualifiers, like /vobs/cc2svn/one@@/main/2/two/main/1.

## Lshistory timestamps are not right

If you do an import which fakes timestamps using setevent, that date
will be shown in lshistory and, worse yet, it will be sorted in that
order.  For example, here is selections of the history of such a vob:

    19961210.164947 mkelem /vobs/cm_test/make-3.79.1/COPYING
    ...
    20010420.164003 mkvob  /vobs/cm_test mkvob
    ...
    20010730.141110 mkelem /view/cc2svn-test/vobs/cm_test/make-3.79.1

I think this will sort itself out, since the COPYING file will
be added to the pending queue until the parent directory is checked
in.

## Missing versions

It appears that removed versions may show up in the history.
To be investigated.

## Mkelem before directory checkin

In ClearCase, an element is created, by definition, before that new
element is added to a directory, and before the new directory version
is checked in.  The time gap could be anywhere from seconds to months.

To deal with this all mkelems have to go on a pending queue, and when
directory checkin shows something getting added, the pending queue
must be checked.

Furthermore this could happen in two ways: the element may only have
/main/0 by the time the directory checkin occurs or it could have
later versions.  We must deal with both.

For example here are two possible mkelem event sequences
 /vobs/cc2svn/foo             mkelem     file element
 /vobs/cc2svn/foo   /main     mkelem     branch
 /vobs/cc2svn/foo   /main/0   mkelem     version
 /vobs/cc2svn/foo   /main/1   checkin    version
 /vobs/cc2svn/.     /main/1   checkin    directory version

Or the last two events could be swapped, which would mean "foo" was
first visible as a zero length file, and then content was added later.

Furthermore, this can happen at any depth... so an entire directory
tree could be added, and the checkins could happen in any sequence.

## Lshistory returns the current element name

If a file (or directory) has been renamed many times, lshistory is
going to provide the name by which it was known in the view where
lshistory was run.  The directory version differences will show the
actual name.  We will need to track things by element oid to keep
this straight.

Currently, I assume that the name returned by lshistory will uniquely
map to an element.  If this is not the case I will need to do a "dump"
on each element to get its actual oid.  Update: actually desc with an
output format of %On will do the trick, but I think that was added to
ClearCase somewhat recently.  I verified it's present in v6, but I
don't have any older machines at hand; hopefully nobody else does
either

## Mklabel records are not preserved

When a label is attached to a version, the record of that "mklabel" is
not preserved.  We will have to look at labels on each version, and
create them on the fly.

## Labels can change

Labels can me moved around at any time.  There is no history of these
actions.  Therefore we can only capture the current state of any given
label.

## Branches are sparse

ClearCase history only records the branch information on elements
which have been checked in on that branch.  This relates to the next
point.

## Branches have no explicit base

ClearCase does not maintain any information about what a branch is
based on.  For example if you branch off the label "FOOBAR", the only
record of this is in the config spec, which is not available to the
converter.

Extra information will need to be provided to the converter so it can
know what label (etc) a given branch is based on.  Otherwise all
branches will be sparse.

## Moving branch bases

Since the relationship between a branch and its "base" is only in the
config spec and not in the vob database, this relationship can be
changed in many ways at any time.  Branches are often "rebased" by
first changing the label in the config spec, and then merging to
resolve any conflicts.

Alternately, the label on which a branch is based on may be changing
over time (q.v.) or the branch could be based on LATEST of one or more
other branches.

It is likely impossible to preserve these actions.

## Branches based on newer labels

Given that a branch's "base" may move to other labels, we cannot fully
create the branch until the base label is extant.  For example if the
branch "foo" is based on the label "FOO\_100", but is rebased many
times and is currently based on "FOO\_200".

## Mapping attributes to properties

It seems that ClearCase attributes would map to Subversion properties,
but it isn't yet clear whether they should be mapped to revision
properties or normal properties, or if it would vary.

## Merge arrows

Preserving merge arrows could be very tricky.

## Multiple branches on an element

It is possible that a given branch type may exist in multiple places
on a version tree.  The default is not to permit this, however it was
default behavior in some very old versions of ClearCase and can still
happen if the branch type is created with the -pbranch option.
I am guessing this is a rare situation.

This means we will need to detect when this happens and mitigate,
perhaps by renaming the later instances of the branch

## Multiple labels on an element

As with branches, it is possible for a label type to be on multiple versions
of an element.

## Hard links

Link Unix, ClearCase has hard links, which basically means that a
given element will be visible in multiple places.  This means an SVN
copy would not be appropriate, since the history would begin diverging
after the copy.  Unless we were to keep track of the "hard links" and
duplicate contents.

# Strategies

The biggest unanswered questions is how to deal with labels and branches.  First lets think about labels as they are simpler.  Here are the options:

1. As checkin events arrive with label(s), that version will be copied
into /tags as appropriate.  This will only be done for files,
directories will be created as needed.

    Having freshly created directories could lead to tree conflicts for
    future merges.

2. Store the versions with labels, and at the end of the run, copy each
file or directory recursively from the top down.  Will need to do a
"replace" operation for all but the top element.

    This could leave us with extra files in the tag.

For branches it is a bit more complicated depending on the label strategy:

1. Much like label option 1, copy files as they are checked into the
branch.  This will leave sparse branches unless supplementary
information is given to the converter.

    Having freshly created directories could lead to tree conflicts for
    future merges.

1a. If supplementary label information is given, the initial mkbranch
    could be preceded by a copy of the tags directory.  But since the tag
    is potentially incomplete, we would need to keep the branch on a list
    of branches needing copies of labeled/tagged files as well.

2. When the initial mkbranch is done, do a copy from the top level
directory corresponding to the version branched.  As other files are
checked into the branch, they will be replaced.

    This could leave us with incorrect unbranched files and possibly extra
    or missing files.

2b. If supplementary label information is given, use it for the initial mkbranch.
    the same concerns apply as with item 1b above.

At this point, it seems that label strategy 1 and branch strategy 1a
are likely the best way to proceed, as they are likely to produce the
most accurate results.

## Directory diff can be out of order

Diff on a directory is presented in alphabetical order, this means some operations can appear out of order.  For example:

    -----[ renamed to ]-----
    < newtest/ 2003-03-07 tfisher
    ---
    > test/ 2003-03-07 tfisher
    -----[ renamed to ]-----
    < test/ 2001-04-16 jparanga
    ---
    > test.old/ 2001-04-16 jparanga

Those two operations have to happen in the reverse order.

However, this behavior has changed as of ClearCase 7.1, which now shows the exact same change like so:

 -----\[ Object changed from \]-----
 < test/ 2001-04-16 jparanga
 ---
 > test/ 2003-03-07 tfisher
 -----\[ deleted \]-----
 < newtest/ 2003-03-07 tfisher
 -----\[ added \]-----
 > test.old/ 2001-04-16 jparanga

I think this avoids the order of operation issue.

TBD, construct test cases and run on both 7.0 and 7.1 and figure out how to mitigate.





