# cc2svn -- ClearCase to Subversion #

NOTE: I am trying to get approval from my company's legal dept to do this as a free software project.  Stay tuned.

## Why A New Tool? ##

All existing ClearCase to Subversion converters suffer from fatal flaws.  The simplest tools simply import static file trees.  The tools that try to convert history do not properly deal with directory versioning, the fatal flaw being that they do not include the "-all" option to "lshistory".  These tools will fail on all but the simplest projects.



