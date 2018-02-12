Coding Conventions
------------------

The `Style Guide for Python Code <http://www.python.org/dev/peps/pep-0008/>`_ (AKA PEP 8) outlines some conventions for developing in Python.  It's a good baseline, though POX does not aim for strict conformance.  Here are some guidelines for writing code in POX, especially if you'd like to have it merged.  Note that in some cases they are in addition to or differ from PEP 8.  Also note that the most important guideline is that code is readable.  Also also note that the code in the repository does not entirely conform with the below guidelines (pull requests that improve consistency are very welcome!).

* *Two spaces for indentation*
* Line wrapping / line length:

 * *Maximum of 79 characters*, though 80 won't kill us.
 * *Use implicit line joining* (e.g. "hanging" parentheses or brackets) rather than the explicit backslash line-continuation character unless the former is very awkward
 * Continue lines beneath the appropriate brace/parenthesis (Lisp style) when that works well. When it doesn't, *my* preference is to indent *a single space*, though I know that drives a lot of Python coders crazy, so I'm hesitant to set a specific rule here as long as it's clear.  The basic rule is that it should be either more or less indentation than usual -- i.e., don't use two spaces for a continued line.  *I am more and more using four spaces.*

* *Two blank lines separate top-level pieces of code with structural significance* (classes, top level functions, etc.). You can use three for separating larger pieces of code (though when one is tempted to do this, it's always a good idea to ask oneself if the two larger pieces of code should be in separate files).   Methods within a class should be one or two, but should be consistent within the particular class.
* *Put a space between function name and opening parenthesis of parameter list*.  Similar for class and superclass.  That is: ":mono:`def foo (bar):`", not ":mono:`def foo(bar):`".
* *Use only new-style classes*.  (This means inherit from object.)
* Docstrings:

 * Either stick to :mono:`""" one line """`, or have the opening and closing :mono:`"""` on lines by themselves.  (The latter is preferred.)
 * The first line should always be a relatively short, standalone description.  Additional lines should be separated from the first by a blank line.

* Naming

 * *Classes should generally be InitialCapped*.
 * *Methods and other attributes should be lower_with_underscores*.  Note that this is currently violated all over the place (though it's getting better).
 * *"Private" members* (which you explicitly don't want others relying on) *should start with an underscore*.
 * *"constants" should be UPPER_WITH_UNDERSCORES*.
 * The *keyword arguments* catch-all variable is *called kw* (against Python convention of :mono:`kwargs`)

Additionally, if you want to get your commits merged, please follow good commit message practice.  For a quick writeup on the subject, see `this blog post <http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html>`_.  In addition, it's nice if the first part of the first line roughly indicates which portion/subsystem the commit relates to when possible, e.g., "openflow" or "forwarding".  For example, :mono:`libopenflow: Fix ofp_match lock/unlock`.  See the existing commit log on github for many examples.
