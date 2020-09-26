.. |prev| replace:: :doc:`orm_data_manipulation`
.. |next| replace:: :doc:`further_reading`

.. include:: tutorial_nav_include.rst

.. _tutorial_orm_related_objects:

Working with Related Objects
=============================

In this section, we will cover one more essential ORM concept, which is that of
how the ORM interacts with mapped classes that refer to other objects. In the
section :ref:`tutorial_declaring_mapped_classes`, the mapped class examples
made use of a construct called :func:`_orm.relationship`.  This construct
defines a linkage between two different mapped classes, or from a mapped class
to itself, the latter of which is called a **self-referential** relationship.

To describe the basic idea of :func:`_orm.relationship`, first we'll review
the mapping in short form, omitting the :class:`_schema.Column` mappings
and other directives::

  from sqlalchemy.orm import relationship
  class User(Base):
      __tablename__ = 'user_account'

      # ... Column mappings

      addresses = relationship("Address", back_populates="user")


  class Address(Base):
      __tablename__ = 'address'

      # ... Column mappings

      user = relationship("User", back_populates="addresses")


Above, the ``User`` class now has an attribute ``User.addresses`` and the
``Address`` class has an attribute ``Address.user``.   The
:func:`_orm.relationship` construct will be used to inspect the table
relationships between the :class:`_schema.Table` objects that are mapped to the
``User`` and ``Address`` classes. As the :class:`_schema.Table` object
representing the
``address`` table has a :class:`_schema.ForeignKeyConstraint` which refers to
the ``user_account`` table, the :func:`_orm.relationship` can determine
unambiguously that there is as :term:`one to many` relationship from
``User.addresses`` to ``User``; one particular row in the ``user_account``
table may be referred towards by many rows in the ``address`` table.

All one-to-many relationships naturally correspond to a :term:`many to one`
relationship in the other direction, in this case the one noted by
``Address.user``. The :paramref:`_orm.relationship.back_populates` parameter,
seen above configured on both :func:`_orm.relationship` objects referring to
the other name, establishes that each of these two :func:`_orm.relationship`
constructs should be considered to be complimentary to each other; we will see
how this plays out in the next section.


Persisting relationships
------------------------

We can start by illustrating what :func:`_orm.relationship` does to instances
of objects.   If we make a new ``User`` object, we can note that there is a
Python list when we access the ``.addresses`` element::

    >>> u1 = User(name='pkrabs', fullname='Pearl Krabs')
    >>> u1.addresses
    []

This object is a SQLAlchemy-specific version of Python ``list`` which
has the ability to track and respond to changes made to it.  The collection
also appeared automatically when we accessed the attribute, even though we never assigned it to the object.
This is similar to the behavior noted at :ref:`tutorial_inserting_orm` where
it was observed that column-based attributes to which we don't explicitly
assign a value also display as ``None`` automatically, rather than raising
an ``AttributeError`` as would be Python's usual behavior.

As the ``u1`` object is still :term:`transient` and the ``list`` that we got
from ``u1.addresses`` has not been mutated (i.e. appended or extended), it's
not actually associated with the object yet, but as we make changes to it,
it will become part of the state of the ``User`` object.

The collection is specific to the ``Address`` class which is the only type
of Python object that may be persisted within it.  Using the ``list.append()``
method we may add an ``Address`` object::

  >>> a1 = Address(email_address="pearl.krabs@gmail.com")
  >>> u1.addresses.append(a1)

At this point, the ``u1.addresses`` collection as expected contains the
new ``Address`` object::

  >>> u1.addresses
  [Address(id=None, email_address='pearl.krabs@gmail.com')]

As we associated the ``Address`` object with the ``User.addresses`` collection
of the ``u1`` instance, another behavior also occurred, which is that the
``User.addresses`` relationship synchronized itself with the ``Address.user``
relationship, such that we can navigate not only from the ``User`` object
to the ``Address`` object, we can also navigate from the ``Address`` object
back to the "parent" ``User`` object::

  >>> a1.user
  User(id=None, name='pkrabs', fullname='Pearl Krabs')

This synchronization occurred as a result of our use of the
:paramref:`_orm.relationship.back_populates` parameter between the two
:func:`_orm.relationship` objects.  This parameter names another
:func:`_orm.relationship` for which complementary attribute assignment / list
mutation should occur.   It will work equally well in the other
direction, which is that if we create another ``Address`` object and assign
to its ``Address.user`` attribute, that ``Address`` becomes part of the
``User.addresses`` collection on that ``User`` object::

  >>> a2 = Address(email_address="pearl@aol.com", user=u1)
  >>> u1.addresses
  [Address(id=None, email_address='pearl.krabs@gmail.com'), Address(id=None, email_address='pearl@aol.com')]

We actually made use of the ``user`` parameter as a keyword argument in the
``Address`` consructor, which is accepted just like any other mapped attribute
that was declared on the ``Address`` class.  It is equivalent to assignment
of the ``Address.user`` attribute after the fact::

  # equivalent effect as a2 = Address(user=u1)
  >>> a2.user = u1

Cascading Objects into the Session
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We now have a ``User`` and two ``Address`` objects that are associated in a
bidirectional structure
in memory, but as noted previously in :ref:`tutorial_inserting_orm` ,
these objects are said to be in the :term:`transient` state until they
are associated with a :class:`_orm.Session` object.

We make use of the :class:`_orm.Session` that's still ongoing, and note that
when we apply the :meth:`_orm.Session.add` method to the lead ``User`` object,
the related ``Address`` object also gets added to that same :class:`_orm.Session`::

  >>> session.add(u1)
  >>> u1 in session
  True
  >>> a1 in session
  True
  >>> a2 in session
  True

The above behavior, where the :class:`_orm.Session` received a ``User`` object,
and followed along the ``User.addresses`` relationship to locate a related
``Address`` object, is known as the **save-update cascade** and is discussed
in detail in the ORM reference documentation at :ref:`unitofwork_cascades`.

The three objects are now in the :term:`pending` state; this means they are
ready to be the subject of an INSERT operation but this has not yet proceeded;
all three objects have no primary key assigned yet, and in addition, the ``a1``
and ``a2`` objects have an attribute called ``user_id`` which refers to the
:class:`_schema.Column` that has a :class:`_schema.ForeignKeyConsraint`
referring to the ``user_account.id`` column; these are also ``None`` as the
objects are not yet associated with a real database row::

    >>> print(u1.id)
    None
    >>> print(a1.user_id)
    None

It's at this stage that we can see the very great utility that the unit of
work process provides; recall in the section :ref:`tutorial_core_insert_values_clause`,
rows were inserted rows into the ``user_account`` and
``address`` tables using some elaborate syntaxes in order to automatically
associate the ``address.user_id`` columns with those of the ``user_account``
rows.  Additionally, it was necessary that we emit INSERT for ``user_account``
rows first, before those of ``address``, since rows in ``address`` are
**dependent** on their parent row in ``user_account`` for a value in their
``user_id`` column.

When using the :class:`_orm.Session`, all this tedium is handled for us and
even the most die-hard SQL purist can benefit from automation of INSERT,
UPDATE and DELETE statements.   When we :meth:`_orm.Session.commit` the
transaction all steps invoke in the correct order, and furthermore the
newly generated primary key of the ``user_account`` row is applied to the
``address.user_id`` column appropriately:

.. sourcecode:: pycon+sql

  >>> session.commit()
  {opensql}INSERT INTO user_account (name, fullname) VALUES (?, ?)
  [...] ('pkrabs', 'Pearl Krabs')
  INSERT INTO address (email_address, user_id) VALUES (?, ?)
  [...] ('pearl.krabs@gmail.com', 6)
  INSERT INTO address (email_address, user_id) VALUES (?, ?)
  [...] ('pearl@aol.com', 6)
  COMMIT


.. _tutorial_select_relationships:

Using Relationships in Queries
-------------------------------

content

.. _tutorial_joining_relationships:

Using Relationships to Join
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The sections :ref:`tutorial_select_join` and
:ref:`tutorial_select_join_onclause` introduced the usage of the
:meth:`_sql.Select.join` and :meth:`_sql.Select.join_from` methods to compose
SQL JOIN clauses.   In order to describe how to join between tables, these
methods either **infer** the ON clause based on the presence of a single
unambiguous :class:`_schema.ForeignKeyConstraint` object within the table
metadata structure that links the two tables, or otherwise we may provide an
explicit SQL Expression construct that indicates a specific ON clause.

When using ORM entities, an additional mechanism is available to help us set up
the ON clause of a join, which is to make use of the :func:`_orm.relationship`
objects that we set up in our user mapping, as was demonstrated at
:ref:`tutorial_declaring_mapped_classes`. The class-bound attribute
corresponding to the :func:`_orm.relationship` may be passed as the **single
argument** to :meth:`_sql.Select.join`, where it serves to indicate both the
right side of the join as well as the ON clause at once::

    >>> print(
    ...     select(Address.email_address).
    ...     select_from(User).
    ...     join(User.addresses)
    ... )
    SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

The presence of an ORM :func:`_orm.relationship` on a mapping is not used
by :meth:`_sql.Select.join` or :meth:`_sql.Select.join_from` if we don't
specify it; it is **not used for ON clause
inference**.  This means, if we join from ``User`` to ``Address`` without an
ON clause, it works because of the :class:`_schema.ForeignKeyConstraint`
between the two mapped :class:`_schema.Table` objects, not because of the
:func:`_orm.relationship` objects on the ``User`` and ``Address`` classes::

    >>> print(
    ...    select(Address.email_address).
    ...    join_from(User, Address)
    ... )
    SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id


Joining to Aliased targets / of_type()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

EXISTS forms / has() / any()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


Common Relationship Operators
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Loader Strategies
-----------------

Selectin Load
^^^^^^^^^^^^^^

Joined Load
^^^^^^^^^^^

Explicit Join + Eager load
^^^^^^^^^^^^^^^^^^^^^^^^^^^
