:tocdepth: 2

.. _`table parameters for reading`: http://cxc.harvard.edu/contrib/asciitable/#commonly-used-parameters-for-read
.. _`asciitable.Basic`: http://cxc.harvard.edu/contrib/asciitable/#asciitable.Basic
.. _`asciitable.Cds`: http://cxc.harvard.edu/contrib/asciitable/#asciitable.Cds
.. _`asciitable.CommentedHeader`: http://cxc.harvard.edu/contrib/asciitable/#asciitable.CommentedHeader
.. _`asciitable.Daophot`: http://cxc.harvard.edu/contrib/asciitable/#asciitable.Daophot
.. _`asciitable.Ipac`: http://cxc.harvard.edu/contrib/asciitable/#asciitable.Ipac
.. _`asciitable.Memory`: http://cxc.harvard.edu/contrib/asciitable/#asciitable.Memory
.. _`asciitable.NoHeader`: http://cxc.harvard.edu/contrib/asciitable/#asciitable.NoHeader
.. _`asciitable.Rdb`: http://cxc.harvard.edu/contrib/asciitable/#asciitable.Rdb
.. _`asciitable.Tab`: http://cxc.harvard.edu/contrib/asciitable/#asciitable.Tab
.. _`asciitable.read()`: http://cxc.harvard.edu/contrib/asciitable/#reading-tables
.. _`asciitable.write()`: http://cxc.harvard.edu/contrib/asciitable/#writing-tables

Asciitable
===========

Astronomers love storing tabular data in human-readable ASCII tables.
Unfortunately there is very little agreement on a standard way to do this,
unlike e.g. FITS.

The `asciitable`_ module is an extensible ASCII table reader and writer that is
designed to handle most formats you will encounter in the wild:

* `asciitable.Basic`_: basic table with customizable delimiters and header configurations
* `asciitable.Cds`_: `CDS format table <http://vizier.u-strasbg.fr/doc/catstd.htx>`_ (also Vizier and ApJ machine readable tables)
* `asciitable.CommentedHeader`_: column names given in a line that begins with the comment character
* `asciitable.Daophot`_: table from the IRAF DAOphot package
* `asciitable.Ipac`_: `IPAC format table <http://irsa.ipac.caltech.edu/applications/DDGEN/Doc/ipac_tbl.html>`_
* `asciitable.Memory`_: table already in memory (list of lists, dict of lists, etc)
* `asciitable.NoHeader`_: basic table with no header where columns are auto-named
* `asciitable.Rdb`_: tab-separated values with an extra line after the column definition line
* `asciitable.Tab`_: tab-separated values

`Asciitable`_ is built on a modular and extensible class structure.  The
basic functionality required for reading or writing a table is largely broken
into independent base class elements so that new formats can be accomodated
by modifying the underlying class methods as needed.

Reading tables
--------------
::

  import asciitable
  table = """
  col1 col2 col3
  1    2    "hi there"
  3    4.2  world"""
  data = asciitable.read(table)

Examine what was returned::

  data

This is an instance of the NumPy `structured array
<http://docs.scipy.org/doc/numpy/user/basics.rec.html#module-numpy.doc.structured_arrays>`_
type, which is an efficient way to manipulate records of tabular data.  It stores
columns of typed data and you can access either a column of data or a row of
data at once::

  data.dtype
  data[1]
  data['col2']

The first and most important argument to the `asciitable.read()`_ function is
the table input.  There is some flexibility here and you can supply any of the following:

  - Name of a file (string)
  - Single string containing all table lines separated by newlines
  - File-like object with a callable read() method
  - List of strings where each list element is a table line

Guessing
^^^^^^^^^^

Even though it seems obvious to a human, parsing this table to get the right
column names, data values and data types is not trivial.  `Asciitable`_ needed
to figure out (or guess):

- Overall table format (DAOphot, CDS, RDB, Basic, etc)
- Column delimiter, e.g. space, comma, tab, vertical bar, etc.
- Column names (which row, maybe preceded by #)
- Quote character (single or double quote)

By default `asciitable`_ will try each format it knows and use the first one
that gives a "reasonable" answer.  The details are in the `Guess table format
<http://cxc.harvard.edu/contrib/asciitable/#guess-table-format>`_ section.  
Sometimes it will fail, e.g.::

  table = """
  col1 & col2
    1  & hi there
    3  & world
  """
  asciitable.read(table)

This gives an ominous looking stack trace, but actually all that happened is
that `asciitable`_ guessed every format it knows and nothing worked.  The
standard set of column delimiters is space, comma, tab, and the vertical bar.
In this case you simply need to give it some help::

  asciitable.read(table, delimiter="&")

The full list of `table parameters for reading`_ includes common options like
``Reader``, ``delimiter``, ``quote_char``, and ``comment``.  

The ``Reader`` option specifies the overall class of table format as documented
in the `Extension Reader Classes
<http://cxc.harvard.edu/contrib/asciitable/#extension-reader-classes>`_ section.

No guessing
^^^^^^^^^^^^

For some tricky tables you will want to disable guessing and explicitly provide
the relevant table format information to the `asciitable.read()`_ function.
A big advantage in this strategy is that `asciitable`_ can then provide more
detailed information if it still fails to parse the table, e.g.::

  asciitable.read(table, guess=False, Reader=asciitable.Basic)

This produces a message (after the stacktrace) that should be a pretty good
clue that `asciitable`_ is using the wrong column delimiter::

  InconsistentTableError: Number of header columns (3) inconsistent with data columns (4) at data line 0
  Header values: ['col1', '&', 'col2']
  Data values: ['1', '&', 'hi', 'there']


Writing
-------

You can write ASCII tables using the `asciitable.write()`_ function.  There is
a lot of flexibility in the format of the `input data
<http://cxc.harvard.edu/contrib/asciitable/#input-data-formats>`_ to be
written:

- Existing ASCII table with metadata
- Data from asciitable.read()
- NumPy structured array or record array
- Sequence of sequences
- Dict of sequences

As a first simple example, read a comma-delimited table and then write it out
as space-delimited::

  table = """
  col1,col2,col3
  1,hello world,2.5
  3,again,5.0"""
  dat = asciitable.read(table)

  import sys
  asciitable.write(dat, sys.stdout)  # print to terminal instead of to file

We can use a different column delimiter::

  asciitable.write(dat, sys.stdout, delimiter='|')

or a different table writer class::

  asciitable.write(dat, sys.stdout, Writer=asciitable.CommentedHeader)

As a final example, imagine you've gathered basic information about 5 galaxies
which you want to write as an ASCII table.  You could just use pure Python file I/O as
shown earlier, but then you may need to be careful about quoting and formatting (and why
rewrite the same code every time when it is already done!).  Instead just use `asciitable`_::

  types = ['barred spiral', 'spiral', 'peculiar (ring)', 'elliptical', 'elliptical']
  redshifts = array([0.024221, 0.132, 0.22, 0.34, 0.45])
  lums = array([1e40, 1.2e40, 2e40, 3e40, 4e40])
  table = {'type': types, 'redshift': redshifts, 'lum': lums}

  asciitable.write(table, 'galaxies.dat', formats={'redshift': '%.5f', 'lum': '%.2e'})
  cat galaxies.dat

.. admonition::  Exercise: scraping table data from the web

   To do this exercise you must first install the `BeautifulSoup
   <http://www.crummy.com/software/BeautifulSoup/>`_ package which will parse
   HTML pages into nice data structures.  **QUIT** your IPython session and from the command line do::
     
     easy_install [--user] BeautifulSoup

   Use the ``--user`` flag if you prefer to install the package into your local
   user area instead of within the system Python installation.

   Now start IPython again and define the following function which converts an
   HTML table to a list of lines with tab-separated values (this will be more
   clear in the next part)::

     from BeautifulSoup import BeautifulSoup
     def html2tsv(html, index=0):
         """Parse the index'th HTML table in ``html``.  Return table as a list of
         tab-separated ASCII table lines"""
         soup = BeautifulSoup(html)
         tables = soup.findAll('table')
         table = tables[index]
         out = []
         for row in table.findAll('tr'):
             colvals = [col.text for col in row.findAll('td')]
             out.append('\t'.join(colvals))
         return out

   Now the exercise is to grab the table data from the `XJET catalog page
   <http://hea-www.harvard.edu/XJET/>`_ into a Python data structure.  You'll
   want to start with::

     import urllib2
     import asciitable
     html = urllib2.urlopen('http://hea-www.harvard.edu/XJET/').read()  # Get web page as string
     table1 = html2tsv(html, 0)   # Parse the first table in the web page
     table2 = html2tsv(html, 1)   # Parse the second table
     table3 = html2tsv(html, 2)   # Parse the third table

   Now examine what you got in the ``table`` variables and use
   `asciitable.read()`_ to parse the right one into a table.  Then plot a
   histogram of the redshift distribution in this sample.

   **HINT**: the table has missing values so include ``fill_values=('', '-1')`` in
   the call to `asciitable.read()`_.  `Asciitable`_ has robust functionality to `replace bad or
   missing values <http://cxc.cfa.harvard.edu/contrib/asciitable/#replace-bad-or-missing-values>`_.

.. raw:: html

   <p class="flip9">Click to Show/Hide Solution</p> <div class="panel9">

The data are in the second table, so do::

  dat = asciitable.read(table2, fill_values=('', '-1'))
  dat.dtype
  dat.dtype.names
  hist(dat['z'], bins=50)

.. image:: xjet_hist.png
   :scale: 50

.. raw:: html

   </div>

.. include:: ../references.rst
