# SQLite3 Native API for Emacs 25+
`sqlite-napi` is a dynamic module for GNU Emacs that provides 
direct access to the core SQLite3 C API.
~~~el
(require 'sqlite3-napi)

(setq dbh (sqlite3-open "person.sqlite3" sqlite-open-readwrite sqlite-open-create))
(sqlite3-exec dbh "create table temp (name text, age integer)")
(sqlite3-exec dbh "begin") ;; begin transaction
(setq stmt (sqlite3-prepare dbh "insert into temp values (?,?)"))
(cl-loop for i from 1 to 10 do
	 (sqlite3-bind-multi stmt (format "name%d" i) i)
	  ;; execute the SQL
	 (sqlite3-step stmt)
	 ;; call reset if you want to bind the SQL to a new set of variables
	 (sqlite3-reset stmt) )
(sqlite3-exec dbh "commit")
(sqlite3-finalize stmt)

(setq stmt (sqlite3-prepare dbh "select * from temp"))
(while (= sqlite-row (sqlite3-step stmt))
  (cl-destructuring-bind (name age) (sqlite3-fetch stmt)
    (message "name: %s, age: %d" name age)))
(sqlite3-finalize stmt)
(sqlite3-close dbh)
~~~
While this module provides only 14 functions (vs 200+ in the C API), it should satisfy most
users' needs.

This is alpha software and might crash your Emacs. Save your work before
trying it out.

## Table of Contents
* [Requirements](#1)
* [Installation](#2)
* [API](#3)
    * [sqlite3-open](#3-1)
    * [sqlite3-close](#3-2)
    * [sqlite3-prepare](#3-3)
    * [sqlite3-finalize](#3-4)
    * [sqlite3-step](#3-5)
    * [sqlite3-changes](#3-6)
    * [sqlite3-reset](#3-7)
    * [sqlite3-last-insert-rowid](#3-8)
    * [sqlite3-get-autocommit](#3-9)
    * [sqlite3-exec](#3-10)
    * [sqlite3-bind-*](#3-11)
    * [sqlite3-bind-multi](#3-12)
    * [sqlite3-column-*](#3-13)
    * [sqlite3-fetch](#3-14)
* [A Note on Garbage Collection](#4)
* [Known Problems](#5)
* [License](#6)
* [Useful Links for Writing Dynamic Modules](#7)

## <a name="1"/> Requirements
- Emacs 25.1 or above, compiled with module support (`--with-modules`)
- sqlite3 library and header file
- A C compiler

It's been tested on macOS and Linux (CentOS).
## <a name="2"/> Installation
~~~sh
$ git co https://github.com/pekingduck/emacs-sqlite3-napi
$ cd emacs-sqlite3-napi
$ make
$ cp sqlite3-napi.el sqlite3-napi-constants.el sqlite3-napi-module.so /your/elisp/load-path/
~~~
A copy of `emacs-module.h` is included in this repo so Emacs source tree
is not needed to build the module.

## <A NAME="3"/> API
An application will typically use sqlite3_open() to create a single database connection during initialization. 

To run an SQL statement, the application follows these steps:

1. Create a prepared statement using sqlite3_prepare().
1. Evaluate the prepared statement by calling sqlite3_step() one or more times.
1. For queries, extract results by calling sqlite3_column() in between two calls to sqlite3_step().
1. Destroy the prepared statement using sqlite3_finalize().
1. Close the database using sqlite3_close().

[SQlite3 constants](https://www.sqlite.org/rescode.html), defined in sqlite3.h, are things such as numeric result codes from various interfaces (ex: `SQLITE_OK`) or flags passed into functions to control behavior (ex: `SQLITE_OPEN_READONLY`).

In elisp they are in lowercase and words are separated by "-" instead of
"_". For example, `SQLITE_OK` would be `sqlite-ok`.

[www.sqlite.org](https://www.sqlite.org) is always a good source of information, especially 
[An Introduction to the SQLite C/C++ Interface](https://www.sqlite.org/cintro.html) and [C/C++ API Reference](https://www.sqlite.org/c3ref/intro.html).

### <a name="3-1"/> sqlite3-open
~~~el
(sqlite3-open "/path/to/data-file" flag1 flag2 ...)
~~~
Open the database file and return a database handle.

This function calls [`sqlite3_open_v2()`](https://www.sqlite.org/c3ref/open.html) internally and raises `'db-error` in case of error.

*flag1*, *flag2*.... will be ORed together.
### <a name="3-2"/> sqlite3-close
~~~el
(sqlite3-close database-handle)
~~~
Close the database file.
### <a name="3-3"/> sqlite3-prepare
~~~el
(sqlite3-prepare database-handle sql-statement)
~~~
Compile the supplied SQL statement and return a statement handle.

This function calls [`sqlite3_prepare_v2()`](https://www.sqlite.org/c3ref/prepare.html) internally and raises 'sql-error (such as invalid SQL statement).
### <a name="3-4"/> sqlite3-finalize
~~~el
(sqlite3-finalize statement-handle)
~~~
Destroy a prepared statement.
### <a name="3-5"/> sqlite3-step
~~~el
(sqlite3-step statement-handle)
~~~
Execute a prepared SQL statement. Some of the return codes are:

`sqlite-done` - the statement has finished executing successfully.

`sqlite-row` - if the SQL statement being executed returns any data, then `sqlite-row` is returned each time a new row of data is ready for processing by the caller. 

### <a name="3-6"/> sqlite3-changes
~~~el
(sqlite3-changes database-handle)
~~~
Return the number of rows modified (for update/delete/insert statements)

### <a name="3-7"/> sqlite3-reset
~~~el
(sqlite3-reset statement-handle)
~~~
Reset a prepared statement. Call this function if you want to re-bind 
the statement to new variables.
### <a name="3-8"/> sqlite3-last-insert-rowid
~~~el
(sqlite3-last-insert-rowid database-handle)
~~~
Retrieve the last inserted rowid (64 bit). 

Notes: Beware that Emacs only supports integers up to 61 bits.
### <a name="3-9"/> sqlite3-get-autocommit
~~~el
(sqlite3-get-autocommit database-handle)
~~~
Return 1 / 0 if auto-commit mode is ON / OFF.
### <a name="3-10"/> sqlite3-exec
~~~el
(sqlite3-exec database-handle sql-statements &optional callback)
~~~
The Swiss Army Knife of the API, you can execute multiple SQL statements
(separated by ";") in a row with just one call.

The callback function, if supplied, is invoked for *each row* and should accept 3
 parameters: 
 1. the first parameter is the number of columns in the current row;
 2. the second parameter is the actual data (as a list strings or nil in case of NULL); 
 3. the third one is a list of column names. 
 
To signal an error condition inside the callback, return `nil`. 
`sqlite3_exec()` will stop the execution and raise 'db-error.

An example of a callback:
~~~el
(defun print-row (ncols data names)
  (cl-loop for i from 0 to (1- ncols) do
           (message "[%d]%s->%s" elt (ncols names i) (elt data i))
           (message "--------------------"))
  t)
  
(sqlite3-exec dbh "select * from table_a; select * from table b"
              #'print-row)
~~~
More examples:
~~~el
;; Update/delete/insert
(sqlite3-exec dbh "delete from table") ;; delete returns no rows

;; Retrieve the metadata of columns in a table
(sqlite3-exec dbh "pragma table_info(table)" #'print-row)

;; Transaction support
(sqlite3-exec dbh "begin")
....
(sqlite3-exec dbh "commit")
...
(sqlite3-exec dbh "rollback")
~~~
### <a name="3-11"/> sqlite3-bind-*
~~~el
(sqlite3-bind-text statement-handle column-no value)
(sqlite3-bind-int64 statement-handle column-no value)
(sqlite3-bind-double statement-handle column-no value)
(sqlite3-bind-null statement-handle column-no)
~~~
The above four functions bind values to a compiled SQL statements.

Please note that column number starts from 1, not 0!
~~~el
(sqlite3-bind-parameter-count statement-handle)
~~~
The above functions returns the number of SQL parameters of a prepared 
statement.
### <a name="3-12"/> sqlite3-bind-multi
~~~el
(sqlite3-bind-multi statement-handle &rest params)
~~~
`sqlite3-bind-multi` binds multiple parameters to a prepared SQL 
statement. It is not part of the official API but is provided for 
convenience.

Example:
~~~el
(sqlite3-bind-multi stmt 1234 "a" 1.555 nil) ;; nil for NULL
~~~
### <a name="3-13"/> sqlite3-column-*
These column functions are used to retrieve the current row
of the result set.

~~~el
(sqlite3-column-count statement-handle)
~~~
Return number of columns in a result set.
~~~e1
(sqlite3-column-type statement-handle column-no)
~~~
Return the type (`sqlite-integer`, `sqlite-float`, `sqlite-text` or
`sqlite-null`) of the specified column. 

Note: Column number starts from 0.
~~~el
(sqlite3-column-text statement-handle column-no)
(sqlite3-column-int64 statement-handle column-no)
(sqlite3-column-double statement-handle column-no)
~~~
The above functions retrieve data of the specified column.

Note: You can call `sqlite3-column-xxx` on a column even 
if `sqlite3-column-type` returns `sqlite-yyy`: the SQLite3 engine will
perform the necessary type conversion.

Example:
~~~el
(setq stmt (sqlite3-prepare dbh "select * from temp"))
(while (= sqlite-row (sqlite3-step stmt))
	(let ((name (sqlite3-column-text stmt 0))
	      (age (sqlite3-column-int64 stmt 1)))
      (message "name: %s, age: %d" name age)))
~~~
### <a name="3-14"/> sqlite3-fetch
~~~el
(sqlite3-fetch statement-handle) ;; returns a list such as (123 56 "Peter Smith" nil)
~~~
`sqlite3-fetch` is not part of the official API but provided for 
convenience. It retrieves the current row as a 
list without having to deal with sqlite3-column-* explicitly.

## <a name="4"/> A Note on Garbage Collection
Since Emacs's garbage collection is non-deterministic, it would be 
a good idea 
to manually free database/statement handles once they are not needed.

## <a name="5"/> Known Problems
- SQLite3 supports 64 bit integers but Emacs integers are only 61 bits.
For integers > 61 bits you can retrieve them as text as a workaround.
- BLOB/ TEXT fields with embedded NULLs are not supported.

## <a name="6"/> License
The code is licensed under the [GNU GPL v3](https://www.gnu.org/licenses/gpl-3.0.html).

## <a name="7"/> Useful Links for Writing Dynamic Modules
- https://phst.github.io/emacs-modules
- http://nullprogram.com/blog/2016/11/05/
