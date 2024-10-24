make github actions go brrr
###########################

This repository exists for a presentation I gave.

The talk walked through some ways to speed up GitHub action workflows
when testing Python projects.


Vocabulary
==========

My talk focused on reducing run times, billable times, and the number of jobs.
This is not everyone's goal (for example, some might try to optimize total duration),
but understanding what these terms mean helps standardize the conversation.

Job
    When a workflow runs, GitHub's interface shows a list of items in the left sidebar.
    Those items are jobs, and they require a runner operating system to spin up
    and perform some individual steps (like checking out the repo, for example).

Total duration
    This value shows up in the workflow run's Summary page.

    It tells you how long it took all of the jobs to run in parallel,
    or "how long did the developer wait to know whether CI passed or failed"?

    My talk did not focus on optimizing this value,
    but almost everything I discussed can help improve it.

Run time
    This value shows up in the workflow run's Usage page.
    It tells you how long it took for each individual job to run.

    In my talk, I focused on the sum of all the run times across all jobs,
    because reducing this value means less resource usage (good for the environment),
    less risk of hitting GitHub quota restrictions (good for your project),
    and less billable time (good for your organization's pocket book).

Billable time
    This value shows up in paid GitHub accounts, on the workflow run's Summary page.

    It's roughly calculated by rounding *each individual job's run time* up
    to the next minute.

    For example, if three jobs take 10, 20, and 30 seconds to run,
    each is rounded up to a full minute and those three minutes are added together
    for a final billable time of 3 minutes.


Performance testing
===================

GitHub workflow run times can vary significantly,
but I designed the Python project to be so onerous to test
that each branch's performance changes should be visible (except branch 3).

In the next section I document each of the branches in the repository
and include a table showing the results of a single workflow run
(or two, to document a cache miss and a cache hit between workflow runs).

The GitHub runners and steps showed considerable jitter.
For example, installing Python 3.8 took between 24 and 51 seconds on macOS,
while Python 3.13 took between 37 seconds and 74 seconds to install on Windows.
Some of the slowest times manifested when testing branch 5,
but I did not re-run the tests.


Repository branches
===================

``main``
--------

Documents the repository.
No branches share ``main`` as an ancestor commit.


``0-base-project``
------------------

This branch contains a basic project.

The project's tox and CI configurations reflect common, documented practices.

*   The tox config just install and runs pytest.
*   The GitHub workflow config uses a matrix
    to create a cross-product of Python versions and runner operating systems.

..  rubric:: Performance

=============== ===============
Category        Branch 0
=============== ===============
Job count       18
Total duration  3m 40s
Run time        29m 7s
Billable time   40 minutes
=============== ===============


``1-all-pythons-per-runner``
----------------------------

This branch introduces a single change:
each runner operating system has *all Python versions* simultaneously installed.
Then, tox simply runs the test suite on all of the Python versions,
just as it would locally.

Put another way: instead of spinning up a Linux runner six times
and install one Python version on each runner,
this branch configures a single Linux runner with six Python versions installed.

..  rubric:: Performance

=============== =============== ===============
Category        Branch 0        Branch 1
=============== =============== ===============
Job count       18              3
Total duration  3m 40s          7m 14s
Run time        29m 7s          14m 50s
Billable time   40 minutes      17 minutes
=============== =============== ===============


``2-cache-tox-installs``
------------------------

This branch configures CI to install tox in a virtual environment and cache it.

This is a modest change in the context of how expensive this project is to test,
but the change introduces caching and demonstrates basic cache-busting.


``3-cache-all-tox-environments``
--------------------------------

This branch configures CI to cache the entire ``.tox/`` directory,
and further expands on how important cache-busting is.

This step may be beneficial for some Python projects, but not for others.
This particular project has roughly 1,300 files to package and install,
and in this branch, tox is configured to install the package in each test environment.
This means that *there are almost 8,000 extra files to cache and restore*
across all six tested Python versions
and the time spent building and restoring the cache reflect this burden.

At the time of writing,
the ``actions/setup-python`` outputs are insufficient for strong cache-busting,
and this is highlighted by the introduction of the `kurtmckee/detect-pythons`_ action.

..  rubric:: Performance

=============== =============== =========================== ===========================
Category        Branch 0        Branch 3 (cache miss)       Branch 3 (cache hit)
=============== =============== =========================== ===========================
Job count       18              3                           3
Total duration  3m 40s          8m 2s                       6m 5s
Run time        29m 7s          17m 10s                     13m 0s
Billable time   40 minutes      18 minutes                  14 minutes
=============== =============== =========================== ===========================


``4-build-a-single-wheel``
--------------------------

This branch makes tox build a single wheel for all of the Pythons to use,
rather than building a single tarball that pip must convert to a wheel
for every tox environment.

This technique may be beneficial for Python projects *that don't require compilation*.

You can read more about this in my blog post, `"Build wheels to go faster"`_.

..  rubric:: Performance

=============== =============== =========================== ===========================
Category        Branch 0        Branch 4 (cache miss)       Branch 4 (cache hit)
=============== =============== =========================== ===========================
Job count       18              3                           3
Total duration  3m 40s          3m 56s                      3m 53s
Run time        29m 7s          8m 41s                      8m 26s
Billable time   40 minutes      10 minutes                  10 minutes
=============== =============== =========================== ===========================


``5-build-shared-wheel``
------------------------

This branch generates a wheel and uploads it as an artifact.
Each test runner then downloads the artifact.

This technique may be useful for projects that have onerous build steps,
particularly if the same artifact(s) can be shared across multiple operating systems.

..  rubric:: Performance

=============== =============== =========================== ===========================
Category        Branch 0        Branch 5 (cache miss)       Branch 5 (cache hit)
=============== =============== =========================== ===========================
Job count       18              4                           4
Total duration  3m 40s          5m 16s                      4m 38s
Run time        29m 7s          10m 41s                     8m 11s
Billable time   40 minutes      13 minutes                  11 minutes
=============== =============== =========================== ===========================


``6-editable-install``
----------------------

This branch configures tox to install the project in editable mode.
Since a wheel is no longer needed, the build job introduced in branch 5 is also removed.

This technique may be beneficial for projects...BUT!
One benefit of building a wheel in your test suite is
*demonstrating that your project can be built and uploaded to PyPI*.
If you completely stop building a wheel in your test suite
there is a risk that you won't discover a packaging misconfiguration
until your users try to download and install your package from PyPI.

..  rubric:: Performance

=============== =============== =========================== ===========================
Category        Branch 0        Branch 6 (cache miss)       Branch 6 (cache hit)
=============== =============== =========================== ===========================
Job count       18              3                           3
Total duration  3m 40s          3m 40s                      1m 53s
Run time        29m 7s          6m 42s                      3m 19s
Billable time   40 minutes      8 minutes                   4 minutes
=============== =============== =========================== ===========================


..  Links
..  =====
..  _kurtmckee/detect-pythons: https://github.com/kurtmckee/detect-pythons
..  _"Build wheels to go faster": https://kurtmckee.org/2023/05/build-wheels-to-go-faster/
