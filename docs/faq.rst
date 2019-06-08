.. _faq:

==========================
Frequently Asked Questions
==========================

Where Can I Report Bugs / Send Feedback / Request Features?
===========================================================

At https://github.com/indygreg/PyOxidizer/issues

Can Python 2.7 Be Supported?
============================

In theory, yes. However, it is considerable more effort than Python 3. And
since Python 2.7 is being deprecated in 2020, in the project author's
opinion it isn't worth the effort.

``No python interpreter found of version 3.*`` Error When Building
==================================================================

This is due to a dependent crate insisting that a Python executable
exist on ``PATH``. Set the ``PYTHON_SYS_EXECUTABLE`` environment
variable to the path of a Python 3.7 executable and try again. e.g.::

   # UNIX
   $ export PYTHON_SYS_EXECUTABLE=/usr/bin/python3.7
   # Windows
   $ SET PYTHON_SYS_EXECUTABLE=c:\python37\python.exe

Why Rust?
=========

``PyOxidizer`` binaries require a *driver* application to interface with
the Python C API and that *driver* application needs to compile to native
code. In the author's opinion, the only appropriate languages for this
were C, C++, and Rust.

Of those 3, the project's author prefers to write new projects in Rust
because it is a superior systems programming language that has built on
lessons learned from decades working with its predecessors. The author
prefers technologies that can detect and eliminate entire classes of bugs
(like buffer overflow and use-after-free) at compile time.

For the non-runtime packaging side of ``PyOxidizer``, pretty much any
programming language would be appropriate. The project's author initially
did prototyping in Python 3 but switched to Rust for synergy with the the
run-time driver and because Rust had *solved* some hard problems such as
a build system that could easily produce statically linked binaries.

Why is the Rust Code... Not Great?
==================================

This is the project author's first real Rust project. Suggestions to improve
the Rust code would be very much appreciated!

Keep in mind that the ``pyoxidizer`` crate is a build-time only
crate and arguably doesn't need to live up to quality standards as
crates containing run-time code. Things like aggressive ``.unwrap()``
usage are arguably tolerable.

The run-time code that produced binaries run (``pyembed``) is held to
a higher standard and is largely ``panic!`` free.

What is the *Magic Sauce* That Makes PyOxidizer Special?
========================================================

There are 2 technical achievements that make ``PyOxidizer`` special.

First, ``PyOxidizer`` consumes Python distributions that were specially
built with the aim of being used for standalone/distributable applications.
These custom-built Python distributions are compiled in such a way that
the resulting binaries have very few external dependencies and run on
nearly every target system. Other tools that produce standalone Python
binaries often rely on an existing Python distribution, which often
doesn't have these characteristics.

Second is the ability to import ``.py``/``.pyc`` files from memory. Most
other self-contained Python applications rely on Python's ``zipimporter``
or do work at run-time to extract the standard library to a filesystem
(typically a temporary directory or a FUSE filesystem like SquashFS). What
``PyOxidizer`` does is expose the ``.py``/``.pyc`` modules data to the
Python interpreter via a Python extension module built-in to the binary.
In addition, the ``importlib._bootstrap_external`` module (which is
*frozen* into ``libpython``) is replaced by a modified version that
defines a custom module importer capable of loading Python modules
from the in-memory data structures exposed from the built-in extension
module.

The custom ``importlib_bootstrap_external`` frozen module trick is
probably the most novel technical achievement of ``PyOxidizer``. Other
Python distribution tools are encouraged to steal this idea!

See the docs in the ``pyembed`` crate for an overview of how the
in-memory import machinery works.

Can Applications Import Python Modules from the Filesystem?
===========================================================

Yes. While the default is to import all Python modules from in-memory
data structures linked into the binary, it is possible to configure
``sys.path`` to allow importing from additional filesystem paths.
Support for importing compiled extension modules is also possible.

What are the Implications of Static Linking?
============================================

Most Python distributions rely heavily on dynamic linking. In addition to
``python`` frequently loading a dynamic ``libpython``, many C extensions
are compiled as standalone shared libraries. This includes the modules
``_ctypes``, ``_json``, ``_sqlite3``, ``_ssl``, and ``_uuid``, which
provide the native code interfaces for the respective non-``_`` prefixed
modules which you may be familiar with.

These C extensions frequently link to other libraries, such as ``libffi``,
``libsqlite3``, ``libssl``, and ``libcrypto``. And more often than not,
that linking is dynamic. And the libraries being linked to are provided
by the system/environment Python runs in. As a concrete example, on
Linux, the ``_ssl`` module can be provided by
``_ssl.cpython-37m-x86_64-linux-gnu.so``, which can have a shared library
dependency against ``libssl.so.1.1`` and ``libcrypto.so.1.1``, which
can be located in ``/usr/lib/x86_64-linux-gnu`` or a similar location
under ``/usr``.

When Python extensions are statically linked into a binary, the Python
extension code is part of the binary instead of in a standalone file.

If the extension code is linked against a static library, then the code
for that dependency library is part of the extension/binary instead of
dynamically loaded from a standalone file.

When ``PyOxidizer`` produces a fully statically linked binary, the code
for these 3rd party libraries is part of the produced binary and not
loaded from external files at load/import time.

There are a few important implications to this.

One is related to security and bug fixes. When 3rd party libraries are
provided by an external source (typically the operating system) and are
dynamically loaded, once the external library is updated, your binary
can use the latest version of the code. When that external library is
statically linked, you need to rebuild your binary to pick up the latest
version of that 3rd party library. So if e.g. there is an important
security update to OpenSSL, you would need to ship a new version of your
application with the new OpenSSL in order for users of your application
to be secure.

Another implication is code compatibility. If multiple consumers try
to use different versions of the same library... TODO