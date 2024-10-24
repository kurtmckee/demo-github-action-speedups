``requirements/``
#################

This directory contains the files that manage test dependencies for the project.


How it's used
=============

Tox is configured to use the exported ``requirements.txt`` files as needed.


How it's updated
================

..  code-block::

    poetry update --directory="requirements/test" --lock
    poetry export --directory="requirements/test" --output="requirements/test/requirements.txt" --without-hashes
