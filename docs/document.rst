Documents
---------

In pyC8, a **document** is a Python dictionary with the following
properties:

* Is JSON serializable.
* May be nested to an arbitrary depth.
* May contain lists.
* Contains the ``_key`` field, which identifies the document uniquely within a
  specific collection.
* Contains the ``_id`` field (also called the *handle*), which identifies the
  document uniquely across all collections within a fabric. This ID is a
  combination of the collection name and the document key using the format
  ``{collection}/{key}`` (see example below).
* Contains the ``_rev`` field. C8 Data Fabric supports MVCC (Multiple Version
  Concurrency Control) and is capable of storing each document in multiple
  revisions. Latest revision of a document is indicated by this field. The
  field is populated by C8 Data Fabric and is not required as input unless you want
  to validate a document against its current revision.

For more information on documents and associated terminologies, refer to
`C8 Data Fabric manual`_. Here is an example of a valid document in "students"
collection:

.. _C8 Data Fabric manual: http://www.macrometa.co

.. testcode::

    {
        '_id': 'students/bruce',
        '_key': 'bruce',
        '_rev': '_Wm3dzEi--_',
        'first_name': 'Bruce',
        'last_name': 'Wayne',
        'address': {
            'street' : '1007 Mountain Dr.',
            'city': 'Gotham',
            'state': 'NJ'
        },
        'is_rich': True,
        'friends': ['robin', 'gordon']
    }

.. _edge-documents:

**Edge documents (edges)** are similar to standard documents but with two
additional required fields ``_from`` and ``_to``. Values of these fields must
be the handles of "from" and "to" vertex documents linked by the edge document
in question (see :doc:`graph` for details). Edge documents are contained in
:ref:`edge collections <edge-collections>`. Here is an example of a valid edge
document in "friends" edge collection:

.. testcode::

    {
        '_id': 'friends/001',
        '_key': '001',
        '_rev': '_Wm3dyle--_',
        '_from': 'students/john',
        '_to': 'students/jane',
        'closeness': 9.5
    }

Standard documents are managed via collection API wrapper:

The simple Way
.. testcode::

    from c8 import C8Client
    # Initialize the C8 client.
    client = C8Client(protocol='https', host='gdn1.macrometa.io', port=443,
                          email='user@example.com', password='hidden')
    
    collection_name = 'students'
    

    # Create a new collection if it does not exist
    if client.has_collection(collection_name):
        print("Collection exists)
    else:
        client.create_collection(name=collection_name)

    # Insert a single document
    document = {'_key': 'Abby', 'age': 22}
    client.insert_document(collection_name=collection_name, document=document)

    # Insert multiple documents
    documents = [
        {'_key': 'John', 'age': 18},
        {'_key': 'Mary', 'age': 21}
    ]
    client.insert_document(collection_name=collection_name, document=documents)

    # Insert data from a csv file
    client.insert_document_from_file(collection_name=collection_name,
                                     csv_filepath="~/data.csv")

    # update document
    client.update_document(collection_name=collection_name,
                 document={'_key': 'John', 'age': 20})
    
    # get all documents
    print(client.get_document_many(collection_name))

    print(client.get_document(collection_name, "John" )_
        
    doc = {'_key': 'John', 'age': 20}
    doc['age'] = 35
    client.replace_document(collection_name, doc)
    
    # delete document
    client.delete_document(collection_name, {'_key': 'John', 'age': 35})



The Object Oriented Way

.. testcode::

    from c8 import C8Client

    # Initialize the C8 Data Fabric client.
    client = C8Client(protocol='https', host='gdn1.macrometa.io', port=443)

    # For the "mytenant" tenant, connect to "test" fabric as tenant admin.
    # This returns an API wrapper for the "test" fabric on tenant 'mytenant'
    # Note that the 'mytenant' tenant should already exist.
    tenant = client.tenant(email='mytenant@example.com', password='hidden')
    fabric = tenant.useFabric('test')
    # Get the API wrapper for "students" collection.
    students = fabric.collection('students')

    # Create some test documents to play around with.
    lola = {'_key': 'lola', 'GPA': 3.5, 'first': 'Lola', 'last': 'Martin'}
    abby = {'_key': 'abby', 'GPA': 3.2, 'first': 'Abby', 'last': 'Page'}
    john = {'_key': 'john', 'GPA': 3.6, 'first': 'John', 'last': 'Kim'}
    emma = {'_key': 'emma', 'GPA': 4.0, 'first': 'Emma', 'last': 'Park'}

    # Insert a new document. This returns the document metadata.
    metadata = students.insert(lola)
    assert metadata['_id'] == 'students/lola'
    assert metadata['_key'] == 'lola'

    # Check if documents exist in the collection in multiple ways.
    assert students.has('lola') and 'john' not in students

    # Retrieve the total document count in multiple ways.
    assert students.count() == len(students) == 1

    # Insert multiple documents in bulk.
    students.import_bulk([abby, john, emma])

    # Retrieve one or more matching documents.
    for student in students.find({'first': 'John'}):
        assert student['_key'] == 'john'
        assert student['GPA'] == 3.6
        assert student['last'] == 'Kim'

    # Retrieve a document by key.
    students.get('john')

    # Retrieve a document by ID.
    students.get('students/john')

    # Retrieve a document by body with "_id" field.
    students.get({'_id': 'students/john'})

    # Retrieve a document by body with "_key" field.
    students.get({'_key': 'john'})

    # Retrieve multiple documents by ID, key or body.
    students.get_many(['abby', 'students/lola', {'_key': 'john'}])

    # Update a single document.
    lola['GPA'] = 2.6
    students.update(lola)

    # Replace a single document.
    emma['GPA'] = 3.1
    students.replace(emma)

    # Delete a document by key.
    students.delete('john')

    # Delete a document by ID.
    students.delete('students/lola')

    # Delete a document by body with "_id" or "_key" field.
    students.delete(emma)

    # Delete multiple documents. Missing ones are ignored.
    students.delete_many([abby, 'john', 'students/lola'])

    # Iterate through all documents and update individually.
    for student in students:
        student['GPA'] = 4.0
        student['happy'] = True
        students.update(student)

You can manage documents via fabric API wrappers also, but only simple
operations (i.e. get, insert, update, replace, delete) are supported and you
must provide document IDs instead of keys:

.. testcode::

    from c8 import C8Client

    # Initialize the C8 Data Fabric client.
    client = C8Client(protocol='https', host='gdn1.macrometa.io', port=443)

    # For the "mytenant" tenant, connect to "test" fabric as tenant admin.
    # This returns an API wrapper for the "test" fabric on tenant 'mytenant'
    # Note that the 'mytenant' tenant should already exist.
    fabric = client.fabric(tenant='mytenant', name='test', username='root', password='hidden')

    # Create some test documents to play around with.
    # The documents must have the "_id" field instead.
    lola = {'_id': 'students/lola', 'GPA': 3.5}
    abby = {'_id': 'students/abby', 'GPA': 3.2}
    john = {'_id': 'students/john', 'GPA': 3.6}
    emma = {'_id': 'students/emma', 'GPA': 4.0}

    collection = fabric.create_collection("students")
    # Insert a new document.
    metadata = collection.insert(lola)
    assert metadata['_id'] == 'students/lola'
    assert metadata['_key'] == 'lola'

    # Check if a document exists.
    assert collection.has(lola) is True

    # Get a document (by ID or body with "_id" field).
    collection.get('students/lola')
    collection.get(abby)

    # Update a document.
    lola['GPA'] = 3.6
    collection.update(lola)

    # Replace a document.
    lola['GPA'] = 3.4
    collection.replace(lola)

    # Delete a document (by ID or body with "_id" field).
    collection.delete('students/lola')

See :ref:`StandardFabric` and :ref:`StandardCollection` for API specification.

When managing documents, using collection API wrappers over fabric API
wrappers is recommended as more operations are available and less sanity
checking is performed under the hood.
