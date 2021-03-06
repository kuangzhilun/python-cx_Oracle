.. _plsqlexecution:

****************
PL/SQL Execution
****************

PL/SQL stored procedures, functions and anonymous blocks can be called from
cx_Oracle.

.. _plsqlproc:

PL/SQL Stored Procedures
------------------------

The :meth:`Cursor.callproc()` method is used to call PL/SQL procedures.

If a procedure with the following definition exists:

.. code-block:: sql

    create or replace procedure myproc (
        a_Value1                            number,
        a_Value2                            out number
    ) as
    begin
        a_Value2 := a_Value1 * 2;
    end;

then the following Python code can be used to call it:

.. code-block:: python

    outVal = cursor.var(int)
    cursor.callproc('myproc', [123, outVal])
    print(outVal.getvalue())        # will print 246

Calling :meth:`Cursor.callproc()` actually generates an anonymous PL/SQL block
as shown below, which is then executed:

.. code-block:: python

    cursor.execute("begin myproc(:1,:2); end;", [123, outval])

See :ref:`bind` for information on binding.


.. _plsqlfunc:

PL/SQL Stored Functions
-----------------------

The :meth:`Cursor.callfunc()` method is used to call PL/SQL functions.

The ``returnType`` parameter for :meth:`~Cursor.callfunc()` is
expected to be a Python type, one of the :ref:`cx_Oracle types <types>` or
an :ref:`Object Type <objecttype>`.

If a function with the following definition exists:

.. code-block:: sql

    create or replace function myfunc (
        a_StrVal varchar2,
        a_NumVal number
    ) return number as
    begin
        return length(a_StrVal) + a_NumVal * 2;
    end;

then the following Python code can be used to call it:

.. code-block:: python

    returnVal = cursor.callfunc("myfunc", int, ["a string", 15])
    print(returnVal)        # will print 38

A more complex example that returns a spatial (SDO) object can be seen below.
First, the SQL statements necessary to set up the example:

.. code-block:: sql

    create table MyPoints (
        id number(9) not null,
        point sdo_point_type not null
    );

    insert into MyPoints values (1, sdo_point_type(125, 375, 0));

    create or replace function spatial_queryfn (
        a_Id     number
    ) return sdo_point_type is
        t_Result sdo_point_type;
    begin
        select point
        into t_Result
        from MyPoints
        where Id = a_Id;

        return t_Result;
    end;
    /

The Python code that will call this procedure looks as follows:

.. code-block:: python

    objType = connection.gettype("SDO_POINT_TYPE")
    cursor = connection.cursor()
    returnVal = cursor.callfunc("spatial_queryfn", objType, [1])
    print("(%d, %d, %d)" % (returnVal.X, returnVal.Y, returnVal.Z))
    # will print (125, 375, 0)

See :ref:`bind` for information on binding.


Anonymous PL/SQL Blocks
-----------------------

An anonymous PL/SQL block can be called as shown:

.. code-block:: python

    var = cursor.var(int)
    cursor.execute("""
            begin
                :outVal := length(:inVal);
            end;""", inVal="A sample string", outVal=var)
    print(var.getvalue())        # will print 15

See :ref:`bind` for information on binding.


Using DBMS_OUTPUT
-----------------

The standard way to print output from PL/SQL is with the package `DBMS_OUTPUT
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-C1400094-18D5-4F36-A2C9-D28B0E12FD8C>`__.  Note, PL/SQL code that uses
``DBMS_OUTPUT`` runs to completion before any output is available to the user.
Also, other database connections cannot access the buffer.

To use DBMS_OUTPUT:

* Call the PL/SQL procedure ``DBMS_OUTPUT.ENABLE()`` to enable output to be
  buffered for the connection.
* Execute some PL/SQL that calls ``DBMS_OUTPUT.PUT_LINE()`` to put text in the
  buffer.
* Call ``DBMS_OUTPUT.GET_LINE()`` or ``DBMS_OUTPUT.GET_LINES()`` repeatedly to
  fetch the text from the buffer until there is no more output.


For example:

.. code-block:: python

    # enable DBMS_OUTPUT

    cursor.callproc("dbms_output.enable")

    # execute some PL/SQL that calls DBMS_OUTPUT.PUT_LINE

    cursor.execute("""
            begin
                dbms_output.put_line('This is the cx_Oracle manual');
                dbms_output.put_line('Demonstrating use of DBMS_OUTPUT');
            end;""")

    # fetch the text that was added by PL/SQL

    chunkSize = 10  # Tune this size for your application

    charArr = connection.gettype("SYS.DBMS_OUTPUT.CHARARR")
    lines = charArr.newobject()

    numLines = cursor.var(int)
    numLines.setvalue(0, chunkSize)

    while numLines.getvalue() == chunkSize:
        cursor.callproc("dbms_output.get_lines", (lines, numLines))
        for line in lines.aslist():
            print(line)

This will produce the following output::

    This is the cx_Oracle manual
    Demonstrating use of DBMS_OUTPUT

An alternative is to call ``DBMS_OUTPUT.GET_LINE()`` once per output line, which
may be much slower:

.. code-block:: python

    textVar = cursor.var(str)
    statusVar = cursor.var(int)
    while True:
        cursor.callproc("dbms_output.get_line", (textVar, statusVar))
        if statusVar.getvalue() != 0:
            break
        print(textVar.getvalue())

Implicit results
----------------

Implicit results permit a Python program to consume cursors returned by a
PL/SQL block without the requirement to use OUT REF CURSOR parameters. The
method :meth:`Cursor.getimplicitresults()` can be used for this purpose. It
requires both the Oracle Client and Oracle Database to be 12.1 or higher.

An example using implicit results is as shown:

.. code-block:: python

    cursor.execute("""
            declare
                cust_cur sys_refcursor;
                sales_cur sys_refcursor;
            begin
                open cust_cur for SELECT * FROM cust_table;
                dbms_sql.return_result(cust_cur);

                open sales_cur for SELECT * FROM sales_table;
                dbms_sql.return_result(sales_cur);
            end;""")

    for implicitCursor in cursor.getimplicitresults():
        for row in implicitCursor:
            print(row)

Data from both the result sets are returned::

    (1, 'Tom')
    (2, 'Julia')
    (1000, 1, 'BOOKS')
    (2000, 2, 'FURNITURE')


Edition-Based Redefinition (EBR)
--------------------------------

Oracle Database's `Edition-Based Redefinition
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-58DE05A0-5DEF-4791-8FA8-F04D11964906>`__ feature enables upgrading of
the database component of an application while it is in use, thereby minimizing
or eliminating down time. This feature allows multiple versions of views,
synonyms, PL/SQL objects and SQL Translation profiles to be used concurrently.
Different versions of the database objects are associated with an "edition".

The simplest way to set an edition is to pass the ``edition`` parameter to
:meth:`cx_Oracle.connect()` or :meth:`cx_Oracle.SessionPool()`:

.. code-block:: python

    connection = cx_Oracle.connect("hr", userpwd, "dbhost.example.com/orclpdb1",
            edition="newsales", encoding="UTF-8")


The edition could also be set by setting the environment variable
``ORA_EDITION`` or by executing the SQL statement:

.. code-block:: sql

    alter session set edition = <edition name>;

Regardless of which method is used to set the edition, the value that is in use
can be seen by examining the attribute :attr:`Connection.edition`. If no value
has been set, the value will be None. This corresponds to the database default
edition ``ORA$BASE``.

Consider an example where one version of a PL/SQL function ``Discount`` is
defined in the database default edition ``ORA$BASE`` and the other version of
the same function is defined in a user created edition ``DEMO``.

.. code-block:: sql

    connect <username>/<password>

    -- create function using the database default edition
    CREATE OR REPLACE FUNCTION Discount(price IN NUMBER) RETURN NUMBER IS
    BEGIN
        return price * 0.9;
    END;
    /

A new edition named 'DEMO' is created and the user given permission to use
editions. The use of ``FORCE`` is required if the user already contains one or
more objects whose type is editionable and that also have non-editioned
dependent objects.

.. code-block:: sql

    connect system/<password>

    CREATE EDITION demo;
    ALTER USER <username> ENABLE EDITIONS FORCE;
    GRANT USE ON EDITION demo to <username>;

The ``Discount`` function for the demo edition is as follows:

.. code-block:: sql

    connect <username>/<password>

    alter session set edition = demo;

    -- Function for the demo edition
    CREATE OR REPLACE FUNCTION Discount(price IN NUMBER) RETURN NUMBER IS
    BEGIN
        return price * 0.5;
    END;
    /

The Python application can then call the required version of the PL/SQL
function as shown:

.. code-block:: python

    connection = cx_Oracle.connect(<username>, <password>, "dbhost.example.com/orclpdb1",
            encoding="UTF-8")
    print("Edition is:", repr(connection.edition))

    cursor = connection.cursor()
    discountedPrice = cursor.callfunc("Discount", int, [100])
    print("Price after discount is:", discountedPrice)

    # Use the edition parameter for the connection
    connection = cx_Oracle.connect(<username>, <password>, "dbhost.example.com/orclpdb1",
            edition = "demo", encoding="UTF-8")
    print("Edition is:", repr(connection.edition))

    cursor = connection.cursor()
    discountedPrice = cursor.callfunc("Discount", int, [100])
    print("Price after discount is:", discountedPrice)

The output of the function call for the default and demo edition is as shown::

    Edition is: None
    Price after discount is:  90
    Edition is: 'DEMO'
    Price after discount is:  50
