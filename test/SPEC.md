# xnumon test framework

`make test`


### Test case interface

A test case is itself an executable located within the subfolder `testcases`.
All c sources will be compiled to executables ending in `.test` automatically.
Test cases can also be scripts of any language with files ending in `.stest`.

A test case executable specifies how it will be compared against produced
xnumon logs by printing one or more test specs to standard output.  Test specs
begin in `spec:`.  Lines not beginning with `spec:` are ignored by the
framework.  Test cases not producing any test specs on standard out are
ignored by the framework, but obviously still need to be executed in order to
find out that there are no specs.

For c test cases, a few defines are available, see `GNUmakefile` for details.
Also, there are convenience includes under `include`.  Global files used by
multiple tests are located in the top-level test directory.

Test cases with a filename beginning in `sudo-` will be executed prepending
`sudo -n`.  This allows to write test cases that need root privileges to
execute successfully.


### Test specs

All test cases should emit at least a `spec:testcase` spec.  The only currently
recognized condition is `returncode`, defining the expected return code of the
test case executable.  The test case will fail if the exit status differs from
the defined value.

```
spec:testcase returncode=0
```

Test cases can further emit log event specs:

-   `spec:xnumon-ops`
-   `spec:xnumon-stats`
-   `spec:image-exec`
-   `spec:process-access`
-   `spec:launchd-add`
-   `spec:socket-bind`
-   `spec:socket-accept`
-   `spec:socket-connect`

These specs tell the test framework to look for a logged event with an
eventcode matching the type and one or more conditions evaluated against the
fields of the event.

For example, the following spec will look for `eventcode=2` with the given
fields.  Fields are compared against exactly what is in the logs, so make sure
to use fully resolved paths for paths containing symlinks, because xnumon logs
fully resolved paths.  The test will succeed only if there is exactly 1 log
record that matches all the conditions.

```
spec:image-exec subject.pid=16323 image.path=/bin/sh image.ident=com.apple.sh image.origin=system script.path=/Users/jdoe/hello-world.sh
```

A spec can also have optional comma-separated flags.  The only currently used
flag is `absent`, which denotes that the test should succeed iff the defined
log record cannot be found.  This is useful when an operation is expected to
fail and should not generate a log entry.

```
spec:absent:image-exec subject.pid=12452 image.path=/bin/sh
```

As a special case, `image.sha256` and other hash fields can be matched against
the hash of the current test case executable by using the asterisk syntax.
Hashes for other files than the test case itself need to be computed in the
test case and emitted as a lowercase hex string.

```
spec:image-exec subject.pid=73553 image.sha256=*
```


### Tipps and tricks

Make sure to flush stdout before performing an exec(2) family syscall,
otherwise the spec may be lost due to buffering.

Make sure to emit the `spec:testcase` as early in the code as possible.  This
makes sure that all abnormal exits of the test case binary are cought,
including crashes, assertion failures and exit as part of error handling.

Try to always match for the pid wherever possible.  Because the log records are
shared between all test cases, matching the `subject.pid` is a good way to make
sure that a test is not picking up on logs produced by a different test case.

Try to wait for subprocesses to propagate errors in child processes to the main
test case process and therefore to the test framework.


### Bugs

Output of the test runner is a bit messy and could be improved.

There is no syntax yet to access array members.

Logs are simply read from `/var/log/xnumon.log`; the framework does not know
how to start and stop xnumon.

The test runner has no notion of failure due to macOS bugs (radar) and failure
due to bugs in the code or test cases.  Test cases that fail due to macOS bugs
should currently be disabled by commenting the respective spec out in order to
allow the test framework to be used for testing code during development.  This
could be fixed by introducing a notion of "this spec can fail due to radar XY".

There should be more and better convenience helpers simplifying common tasks
in test cases and reducing code duplication.

The test framework only reduces log volume based on timestamps.  It could also
reduce based on process ancestry to itself.
