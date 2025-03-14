#################
Advanced features
#################


****************************************
Static websites hosted in object storage
****************************************

Using the object storage service, it is possible to host simple websites that
contain only static content from within a container. To do so, you will need to
prepare some resources before we begin.

You will need to have the following ready before you go further:

- An understanding of object storage ACLs (discussed in the "managing access"
  section of these docs)
- Your standard html and css files needed for styling your website.

First, set up a container and configure the read ACL to allow read access and
optionally allow files to be listed.

.. code-block:: bash

  $ swift post con0
  $ swift post -r '.r:*,.rlistings' con0

To confirm the ACL settings, or any other metadata settings that follow
you can run the following command:

.. code-block:: bash

  $ swift stat con0
           Account: AUTH_b24e9ee3447e48eab1bc99cb894cac6f
         Container: con0
           Objects: 3
             Bytes: 35354
          Read ACL: .r:*,.rlistings
         Write ACL:
           Sync To:
          Sync Key:
     Accept-Ranges: bytes
        X-Trans-Id: tx54e1341d5fd74634b19c5-005906aaf6
            Server: nginx/1.10.1
       X-Timestamp: 1493608620.58190
  X-Storage-Policy: Policy-0
      Content-Type: text/plain; charset=utf-8

Next, you will have to upload the files you wish to host:

.. code-block:: bash

  $ swift upload con0 index.html error.html image.png styles.css

It is possible to allow listing of all files in the container by enabling
``web-listings``. It is also possible to style these listings using a separate
CSS file to the one you would use to style the actual website.

Upload the CSS file and enable the web listing and styling for the listing.

.. code-block:: bash

  swift upload con0 listing.css
  swift post -m 'web-listings: true' con0
  swift post -m 'web-listings-css:listings.css' con0

You should now be able to view the files in the container by visiting
the container's URL, where %AUTH_ID% & %container_name% are replaced by
the values specific to your project.

https://object-storage.nz-por-1.catalystcloud.io:443/v1/%AUTH_ID%/%container_name%/

To enable the container to work as a full website, it is also necessary to
enable the index, and optionally, the error settings:

.. code-block:: bash

  swift post -m 'web-index:index.html' con0
  swift post -m 'web-error:error.html' con0

You should now be able to view the index file as a website.

https://object-storage.nz-por-1.catalystcloud.io:443/v1/%AUTH_ID%/%container_name%/

*****************
Object versioning
*****************

.. _object-versioning:

This provides a means by which multiple versions of your content can be stored
allowing for recovery from unintended overwrites.

To enable object versioning for a container, you must specify an **archive
container** that will retain non-current versions via either the
``X-Versions-Location`` or ``X-History-Location header``. These two headers
enable two distinct modes of operation which we will discuss shortly.

First, you need to create an archive container to store the older versions of
your objects:

.. code-block:: bash

  $ curl -i -X PUT -H "X-Auth-Token: $token" $storageURL/archive

Now that you have an archive, you can create another container to store your
objects in. This is where you are able to choose which header type to use for
your container.

* If you use the *X-History-Location* header, then object DELETE requests will
  copy the current version to the archive container and remove the original
  from the versioned container.

* If you instead use *X-Versions-Location*, then object DELETE requests will
  restore the most-recent version from the archive container, overwriting the
  current version in your regular container.

For this example, we are going to use the ``X-Versions-Location`` header:

.. code-block:: bash

  $ curl -i -X PUT -H "X-Auth-Token: $token" -H 'X-Versions-Location: archive' $storageURL/my-container
  HTTP/1.1 201 Created
  Server: nginx/1.10.1
  Date: Mon, 05 Dec 2016 23:50:00 GMT
  Content-Type: text/html; charset=UTF-8
  Content-Length: 0
  X-Trans-Id: txe6d2f4e289654d02a7329-005845fd28

Once the ``X-Versions-Location`` header has been applied to the container, any
changes to objects in the container automatically result in a copy of the
original object being placed in the archive container. The backed up version
will have the following format:

.. code-block::

  <length><object_name>/<timestamp>

Where <length> is the length of the object name ( as a three character zero
padded hex number ), <object_name> is the original object name and <timestamp>
is the unix timestamp of the original file creation.

<length> and <object_name> are then combined to make a new container
(pseudo-folder in the dashboard) with the backed up object stored within using
the timestamp as its name.

.. note::

  You must UTF-8-encode and then URL-encode the container name before you
  include it in the X-Versions-Location header.

If you list your current containers, you can see you now have two empty
containers.

.. code-block:: bash

  $ openstack container list --long

  +--------------+-------+-------+
  | Name         | Bytes | Count |
  +--------------+-------+-------+
  | archive      |     0 |     0 |
  | my-container |     0 |     0 |
  +--------------+-------+-------+

If you upload a sample file into my-container, you can see the confirmation of
this operation. This includes the etag, which is an MD5 hash of the object's
contents.

.. code-block:: bash

  $ openstack object create my-container file1.txt

  +-----------+--------------+----------------------------------+
  | object    | container    | etag                             |
  +-----------+--------------+----------------------------------+
  | file1.txt | my-container | 2767104ea585e1a98axxxxxxddeeae4a |
  +-----------+--------------+----------------------------------+

Now if the original file is modified and uploaded to the same container, you
will get a successful confirmation, except this time you get a new etag, as the
contents of the file have changed.

.. code-block:: bash

  $ openstack object create my-container file1.txt

  +-----------+--------------+----------------------------------+
  | object    | container    | etag                             |
  +-----------+--------------+----------------------------------+
  | file1.txt | my-container | 9673f4c3efc2ee8dd9exxxxxx60c76c4 |
  +-----------+--------------+----------------------------------+

If you show the containers again, you can see now that even though you only
uploaded the file into my-container, you now also have a file present in the
archive container.

.. code-block:: bash

  $ os container list --long

  +--------------+-------+-------+
  | Name         | Bytes | Count |
  +--------------+-------+-------+
  | archive      |    70 |     1 |
  | my-container |    73 |     1 |
  +--------------+-------+-------+

Further investigation of the archive container reveals that you have a new
object, which was created automatically, and named in accordance with the
convention outlined above.

.. code-block:: bash

  $ openstack object list archive

  +-------------------------------+
  | Name                          |
  +-------------------------------+
  | 009file1.txt/1480982072.29403 |
  +-------------------------------+


*************
Temporary URL
*************

This is a means by which a temporary URL can be generated, to allow
unauthenticated access to a Swift object at a given path. The
access is via the given HTTP method (e.g. GET, PUT) and is valid until an
expiry time is reached.

There are several items of information needed to create a temporary URL:

- The temporary URL secret key for your account, which is used to encode details in the temporary URL.
- The URL path to the object you want to share.
- When you want to the temporary URL to expire.
- The HTTP method allowed to access the object, such as "GET" or "PUT"

The swift command line tool
===========================

The swift command line tool is the recommended way to generate a temporary URL.
The syntax for the temp-url creation command is:

``$ swift tempurl [command-option] [method] [time] [path] [key]``

This generates a temporary URL allowing unauthenticated access to the Swift
object at the given path.

The expiry time can be expressed as valid for the given number of seconds from
now or if the optional --absolute argument is provided, seconds is instead
interpreted as a Unix timestamp at which the URL should expire.  The expiry
time can also be given as a date time string in the format 'YYYY-mm-dd',
'YYYY-mm-ddTHH:MM:ss' or 'YYYY-mm-ddTHH:MM:ssZ' (the last is in UTC).

For example:

.. code-block:: bash

  $ swift tempurl GET $(date -d "Jan 1 2023" +%s)  /v1/AUTH_foo/bar_container/quux.md \
    my_secret_tempurl_key --absolute

- AUTH_foo is the object store account id
- sets the expiry using the absolute method to be Jan 1 2023
- for the object : quux.md
- in the nested container structure : bar_container/quux.md
- with key : my_secret_tempurl_key

Or by passing the expiry date:

.. code-block:: bash

  $ swift tempurl GET 2023-01-01 /v1/AUTH_foo/bar_container/quux.md my_secret_tempurl_key


Creating temporary URLs in the Catalyst Cloud
=============================================

Currently, there are two methods available for the creation of temporary URLs:
through the use of the swift command line tool or by generating the temporary URL
using your own code.

For the following examples, we will create a URL that will be valid for 600 seconds and
provide access to the object "file2.txt" that is located in the container
"my-container".

Set the temporary URL secret key for your project
-------------------------------------------------

Firstly you need to associate a secret key with your object store account for your project.

.. code-block:: bash

  $ openstack object store account set --property Temp-Url-Key='testkey'

You can then confirm the details of the key.

.. code-block:: bash

  $ openstack object store account show

  +------------+---------------------------------------+
  | Field      | Value                                 |
  +------------+---------------------------------------+
  | Account    | AUTH_b24e9ee3447e48eab1bc99cb894cac6f |
  | Bytes      | 128                                   |
  | Containers | 4                                     |
  | Objects    | 8                                     |
  | properties | Temp-Url-Key='testkey'                |
  +------------+---------------------------------------+

Note the value for "Account" above, this value identifies the project that the
object is in and is used as part of path in the URL to the object.

It is recommended that the key be at least 32 characters long.

Determine the path to the object
--------------------------------

The path to the object includes the URL path for the object storage API URL for your
project, the container name and the object name.

Suppose that the object storage API URL is ``https://object-storage.nz-por-1.catalystcloud.io:443/v1/AUTH_b24e9ee3447e48eab1bc99cb894cac6f``.

Then the path we are interesting in is "/v1/AUTH_b24e9ee3447e48eab1bc99cb894cac6f".

If container is called "my-container" and the object is called "file2.txt" then
the path to the object will be "/v1/AUTH_b24e9ee3447e48eab1bc99cb894cac6f/my-container/file2.txt".

For more details on the API endpoints see :doc:`/sdks-and-toolkits/apis`

Using the swift command line tool to generate the temporary URL
---------------------------------------------------------------

Then, using the syntax outlined above, you can create a temporary URL to access
an object residing in the object store.

.. code-block:: bash

  $ swift tempurl GET 600 /v1/AUTH_b24e9ee3447e48eab1bc99cb894cac6f/my-container/file2.txt "testkey" \
    /v1/AUTH_b24e9ee3447e48eab1bc99cb894cac6f/my-container/file2.txt?temp_url_sig=2dbc1c2335a53d5548dab178d59ece7801e973b4&temp_url_expires=1483990005

You can test this using cURL and appending the generated URL to the Catalyst
Cloud's object storage base URL "https://object-storage.nz-por-1.catalystcloud.io:443". If
it is successful, the request should return the contents of the object.

.. code-block:: bash

  $ curl -i "https://object-storage.nz-por-1.catalystcloud.io:443/v1/AUTH_b24e9ee3447e48eab1bc99cb894cac6f/my-container/file2.txt?temp_url_sig=2dbc1c2335a53d5548dab178d59ece7801e973b4&temp_url_expires=1483990005"
  HTTP/1.1 200 OK
  Server: nginx/1.10.1
  Date: Mon, 09 Jan 2017 19:22:05 GMT
  Content-Type: text/plain
  Content-Length: 501
  Accept-Ranges: bytes
  Last-Modified: Mon, 09 Jan 2017 19:18:47 GMT
  Etag: 137eed1d424a588318xxxxxx5433594a
  X-Timestamp: 1483989526.71129
  Content-Disposition: attachment; filename="file2.txt"; filename*=UTF-8''file2.txt
  X-Trans-Id: tx9aa84268bd984358b6afe-005873e2dd

  "For those who have seen the Earth from space, and for the hundreds and perhaps thousands more who will, the experience most certainly changes your perspective. The things that we share in our world are far more valuable than those which divide us." "For those who have seen the Earth from space, and for the hundreds and perhaps thousands more who will, the experience most certainly changes your perspective. The things that we share in our world are far more valuable than those which divide us."

You could also access the object by taking the same URL that you passed to cURL
and pasting it into a web browser.

Programmatically generate the temporary URL
-------------------------------------------

You are also able to generate your temporary URLs programmatically using a small bit
of code, here is an example in Python 3:

.. code-block:: python3

  import hmac
  from hashlib import sha1
  import time

  # The HTTP method we want to allow for the temp URL
  method = 'GET'
  # The time the temp URL will expire, in seconds since the epoch
  expires = time.now() + 600

  # the API endpoint for the object store for your account see the API access page on the dashboard
  object_store_api_url = 'https://object-storage.nz-por-1.catalystcloud.io:443'
  # the path to the object to share
  object_path = '/v1/AUTH_b24e9ee3447e48eab1bc99cb894cac6f/my-container/file2.txt'
  # The object store temp URL key
  key = 'testkey'

  # The method, expires and object_path is encrypted using the key
  hmac_body = "{}\n{}\n{}".format(method, expires, object_path)
  sig = hmac.new(key.encode('utf-8'), hmac_body.encode('utf-8'), sha1).hexdigest()

  url = '{base_url}{path}?temp_url_sig={sig}&temp_url_expires={expires}'.format(base_url=object_store_api_url,
                                                                                path=object_path, sig=sig,
                                                                                expires=expires)
  print(url)

The above code is based on the code for the swift tool:
https://opendev.org/openstack/python-swiftclient/src/branch/stable/train/swiftclient/utils.py#L71

**************************
Working with large objects
**************************

Typically, the size of a single object cannot exceed 5GB. It is possible,
however, to use several smaller objects to break up the large object. When this
approach is taken, the resulting large object is made out of two types of
objects:

- **Segment Objects** which store the actual content. You need to split your
  content into chunks and then upload each piece as its own segment object.

- A **manifest object** then links the segment objects into a single logical
  object. To download the object, you download the manifest. Object storage
  then concatenates the segments and returns the contents.

There are tools available, both GUI and CLI, that will handle the segmentation
of large objects for you. For all other cases, you must manually split the
oversized files and manage the manifest objects yourself.

Using the Swift command line tool
=================================

The Swift tool which is included in the `python-swiftclient`_ library is
capable of handling oversized files and gives you the choice of
using either``static large objects (SLO)`` or``dynamic large objects (DLO)``,
which will be explained in more detail later.

.. _python-swiftclient: http://github.com/openstack/python-swiftclient

Before getting in to the distinctions between SLO and DLO, here are two
examples of how to upload a large object to an object storage
container using the Swift tool. To keep the output brief, a 512MB file
is used in the example.

example 1 : DLO
---------------

The default mode for the tool is the ``dynamic large object`` type, so in this
example, the only other parameter that is required is the segment size.
The ``-S`` flag is used to specify the size of each chunk, in this case
104857600 bytes (100MB).

.. code-block:: bash

  $ swift upload mycontainer -S 104857600 large_file
  large_file segment 5
  large_file segment 0
  large_file segment 4
  large_file segment 3
  large_file segment 1
  large_file segment 2
  large_file

example 2 : SLO
---------------

In the second example, the same segment size as above is used, but you specify
that the object type must now be the ``static large object`` type.

.. code-block:: bash

  $ swift upload mycontainer --use-slo -S 104857600 large_file
  large_file segment 5
  large_file segment 1
  large_file segment 4
  large_file segment 0
  large_file segment 2
  large_file segment 3
  large_file

|

Both of these approaches will successfully upload your large file into
object storage. The file would be split into 100MB segments which are
uploaded in parallel. Once all the segments are uploaded, the manifest file
will be created so that the segments can be downloaded as a single
object.

The Swift tool uses a strict convention for its segmented object support.
All segments that are uploaded are placed into a second container that has
``_segments`` appended to the original container name, in this case it would be
mycontainer_segments. The segment names follow the format of
``<name>/<timestamp>/<object_size>/<segment_size>/<segment_name>``.

If you check on the segments created in example 1, you can see this:

.. code-block:: bash

  $ swift list mycontainer_segments
  large_file/1500505735.549995/536870912/104857600/00000000
  large_file/1500505735.549995/536870912/104857600/00000001
  large_file/1500505735.549995/536870912/104857600/00000002
  large_file/1500505735.549995/536870912/104857600/00000003
  large_file/1500505735.549995/536870912/104857600/00000004
  large_file/1500505735.549995/536870912/104857600/00000005


In the above example, it will upload all the segments into a second container
named test_container_segments. These segments will have names like
large_file/1290206778.25/21474836480/00000000,
large_file/1290206778.25/21474836480/00000001, etc.

The main benefit for using a separate container is that the main container
listings will not be polluted with all the segment names. The reason for using
the segment name format of <name>/<timestamp>/<size>/<segment> is so that
an upload of a new file with the same name will not overwrite the contents of
the first until the last moment when the manifest file is updated.


Swift will manage these segment files for you, deleting old segments on deletes
and overwrites, etc. You can override this behavior with the
``--leave-segments`` option if desired; this is useful if you want to have
multiple versions of the same large object available.

*********************************************************
Dynamic Large Objects (DLO) vs Static Large Objects (SLO)
*********************************************************

The main difference between the two object types is to do with the associated
manifest file that describes the overall object structure within Swift.

In both of the examples above, the file would be split into 100MB chunks
and uploaded. This can happen concurrently if desired. Once the segments
are uploaded, it is then necessary to create a manifest file to describe
the object and allow it to be downloaded as a single file. When using
Swift, the manifest files are created for you.

The manifest for the ``DLO`` is an empty file and all segments must be
stored in the same container, though depending on the object store
implementation the segments, as mentioned above, may go into a container
with '_segments' appended to the original container name. It also works
on the assumption that the container will eventually be consistent.

For ``SLO`` the difference is that a user-defined manifest file describing
the object segments is required. It also does not rely on eventually
consistent container listings to do so. This means that the segments can
be held in different container locations. The fact that once all files can't
then change is the reason why these are referred to as 'static' objects.


A more manual approach
======================

While the Swift tool is certainly handy as it handles a lot of the underlying
file management tasks required to upload files into object storage, the same
can be achieved by more manual means.

Here is an example using standard linux commandline tools such as
``split`` and ``curl`` to perform a dynamic large object file upload.

The file 'large_file' is broken into 100MB chunks which are prefixed with
'split-'

.. code-block:: bash

  $ split --bytes=100M large_file split-


The upload of these segments is then handled by cURL. See :ref:`using
curl<s3-api-documentation>` for more information on how to do this.

.. _using curl: http://docs.catalystcloud.io/object-storage.html#using-curl

The first cURL command creates a new container. The next two upload the two
segments created previously, and finally, a zero byte file is created for the
manifest.

.. code-block:: bash

  $ curl -i $storageURL/lgfile -X PUT -H “X-Auth-Token:$token"
  $ curl -i $storageURL/lgfile/split_aa -X PUT -H "X-Auth-Token:$token" -T split-aa
  $ curl -i $storageURL/lgfile/split_ab -X PUT -H "X-Auth-Token:$token" -T split-ab
  $ curl -i -X PUT -H "X-Auth-Token: $token" -H "X-Object-Manifest:lgfile/split" -H "Content-Length: 0"  $storageURL/lgfile/manifest/1gb_sample.txt

A similar approach can also be taken to use the SLO type, but this is a lot
more involved. A detailed description of the process can be seen `here`_


.. _here: https://docs.openstack.org/swift/latest/overview_large_objects.html#module-swift.common.middleware.slo
