
The python API follows C API exactly and you have full access to all
data structures.  It's not that complicated but it it is not very
pythonic. Future versions of the API will likely include some object
wrapper to make it more simple and prevent bugs.

You should get nearly 400,000 checks per second using the python API.

Install the python module
-------------------------

```
git clone https://github.com/client9/libinjection.git
cd libinjection/python
python setup.py install
```

Note, if it complains about SWIG missing, then  TBD... you
don't actually need SWIG installed but...  TBD.

Using libinjection
--------------------------

```python
from libinjection import *

# create the data object
s = sqli_state()

# initialize it
# last arg should be 0 for now.
astr = "1 UNION ALL SELECT 1"
sqli_init(s, astr, 0)

# check!  return 1 = is sql, 0 is not sqli
issqli = is_sqli(s)

# you might wish to turn it into a bool by doing
# issqli = bool(is_sqli(s))

# if it's true, then you can get the SQLi fingerprint
# by looking in the state object

print s.fingerprint

# reset and do it again
#   again last argument should be 0
astr = "here's a new input"
sqli_reset(s, astr, 0)

is_sqli(s)

```


warning
----------------------------

This is probably a bug, but for now, the input string must not change
or get destroyed between the sqli_init (or sqli_reset) call and when
you call is_sqli.  While annoying, this prevents having to make a
string copy, which would slow things down.

For instance:

```python
s = sqli_state()

astr = "something"
sqli_init(s, astr, 0)

astr = None

is_sqli(s)
```

If python crashes when using libinjection, this is why.


Advanced Callbacks
----------------------------

By default, libinjection's SQLi module has a built-in list of
known-sql tokens and known SQLi fingerprints.  Using the callback
functionality, you are able to replace and change this list.

The main use-case is to rapidly update the SQL token and SQLi
fingerprint list without having the end-user do a full libinjection
upgrade.

Unfortunately, by using the callbacks, the performance drops by over
50%.  However it may be useful in some situations.

The current unit-test driver uses this callback, and it's maybe
easiest explaining it with code.

First a small program `json2python.py` coverts the raw JSON file
containing all the data the C program uses into python.  To
use it

```
cd libinjection/python
make words.py
```

The output file `words.py` starts like this:

```
import libinjection

def lookup(state, stype, keyword):
    keyword = keyword.upper()
    if stype == libinjection.LOOKUP_FINGERPRINT:
        if keyword in fingerprints and libinjection.sqli_not_whitelist(state):
            return 'F'
        else:
            return chr(0)
    return words.get(keyword, chr(0))
```

The input is:

* `sqli_state` object
* a lookup type.  The only value that matter is `libinjection.LOOKUP_FINGERPRINT`
* the word to look-up

The output is a string of length 1 (a single character).  If it's "\0" (i.e. `chr(0)`), then
it means "nothing matched" or "not found"

This is likely to change in V4 of libinjection to nicer but, today,
this is what it is.

Of note is this line:

```python
if keyword in fingerprints and libinjection.sqli_not_whitelist(state):
```

The C version of the code contains some logic to try and remove false
positives from a few fingerprint types.  You can reuse this logic in
python by calling `sqli_not_whitelist` which returns `true` if it's
SQLi.  For more control it would be best to re-implement this logic in
python (and probably faster too).

Also note: if your callback throws exceptions or doesn't follow the
API exactly, python is likely to crash.