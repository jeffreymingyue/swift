=================
Object Encryption
=================

Swift supports the optional encryption of object data at rest on storage nodes.
The encryption of object data is intended to mitigate the risk of users' data
being read if an unauthorised party were to gain physical access to a disk.

.. note::

    Swift's data-at-rest encryption accepts plaintext object data from the
    client, encrypts it in the cluster, and stores the encrypted data. This
    protects object data from inadvertently being exposed if a data drive
    leaves the Swift cluster. If a user wishes to ensure that the plaintext
    data is always encrypted while in transit and in storage, it is strongly
    recommended that the data be encrypted before sending it to the Swift
    cluster. Encrypting on the client side is the only way to ensure that the
    data is fully encrypted for its entire lifecycle.

Encryption of data at rest is implemented by middleware that may be included in
the proxy server WSGI pipeline. The feature is internal to a Swift cluster and
not exposed through the API. Clients are unaware that data is encrypted by this
feature internally to the Swift service; internally encrypted data should never
be returned to clients via the Swift API.

The following data are encrypted while at rest in Swift:

* Object content i.e. the content of an object PUT request's body
* The entity tag (ETag) of objects that have non-zero content
* All custom user object metadata values i.e. metadata sent using
  X-Object-Meta- prefixed headers with PUT or POST requests

Any data or metadata not included in the list above are not encrypted,
including:

* Account, container and object names
* Account and container custom user metadata values
* All custom user metadata names
* Object Content-Type values
* Object size
* System metadata

.. note::

    This feature is intended to provide `confidentiality` of data that is at
    rest i.e. to protect user data from being read by an attacker that gains
    access to disks on which object data is stored.

    This feature is not intended to prevent undetectable `modification`
    of user data at rest.

    This feature is not intended to protect against an attacker that gains
    access to Swift's internal network connections, or gains access to key
    material or is able to modify the Swift code running on Swift nodes.

.. _encryption_deployment:

------------------------
Deployment and operation
------------------------

Encryption is deployed by adding two middleware filters to the proxy
server WSGI pipeline and including their respective filter configuration
sections in the `proxy-server.conf` file. :ref:`Additional steps
<container_sync_client_config>` are required if the container sync feature is
being used.

The `keymaster` and `encryption` middleware filters must be to the right of all
other middleware in the pipeline apart from the final proxy-logging middleware,
and in the order shown in this example::

  <other middleware> keymaster encryption proxy-logging proxy-server

  [filter:keymaster]
  use = egg:swift#keymaster
  encryption_root_secret = your_secret

  [filter:encryption]
  use = egg:swift#encryption
  # disable_encryption = False

See the `proxy-server.conf-sample` file for further details on the middleware
configuration options.

The keymaster config option ``encryption_root_secret`` MUST be set to a value
of at least 44 valid base-64 characters before the middleware is used and
should be consistent across all proxy servers. The minimum length of 44 has
been chosen because it is the length of a base-64 encoded 32 byte value.

.. note::

    The ``encryption_root_secret`` option holds the master secret key used for
    encryption.  The security of all encrypted data critically depends on this
    key and it should therefore be set to a high-entropy value. For example, a
    suitable ``encryption_root_secret`` may be obtained by base-64 encoding a
    32 byte (or longer) value generated by a cryptographically secure random
    number generator.

    The ``encryption_root_secret`` value is necessary to recover any encrypted
    data from the storage system, and therefore, it must be guarded against
    accidental loss. Its value (and consequently, the proxy-server.conf file)
    should not be stored on any disk that is in any account, container or
    object ring.

One method for generating a suitable value for ``encryption_root_secret`` is to
use the ``openssl`` command line tool::

    openssl rand -base64 32

Once deployed, the encryption filter will by default encrypt object data and
metadata when handling PUT and POST requests and decrypt object data and
metadata when handling GET and HEAD requests. COPY requests are transformed
into GET and PUT requests by the :ref:`copy` middleware before reaching the
encryption middleware and as a result object data and metadata is decrypted and
re-encrypted when copied.

Upgrade Considerations
----------------------

When upgrading an existing cluster to deploy encryption, the following sequence
of steps is recommended:

#. Upgrade all object servers
#. Upgrade all proxy servers
#. Add keymaster and encryption middlewares to every proxy server's middleware
   pipeline with the encryption ``disable_encryption`` option set to ``True``
   and the keymaster ``encryption_root_secret`` value set as described above.
#. If required, follow the steps for :ref:`container_sync_client_config`.
#. Finally, change the encryption ``disable_encryption`` option to ``False``

Objects that existed in the cluster prior to the keymaster and encryption
middlewares being deployed are still readable with GET and HEAD requests. The
content of those objects will not be encrypted unless they are written again by
a PUT or COPY request. Any user metadata of those objects will not be encrypted
unless it is written again by a PUT, POST or COPY request.

Disabling Encryption
--------------------

Once deployed, the keymaster and encryption middlewares should not be removed
from the pipeline. To do so will cause encrypted object data and/or metadata to
be returned in response to GET or HEAD requests for objects that were
previously encrypted.

Encryption of inbound object data may be disabled by setting the encryption
``disable_encryption`` option to ``True``, in which case existing encrypted
objects will remain encrypted but new data written with PUT, POST or COPY
requests will not be encrypted. The keymaster and encryption middlewares should
remain in the pipeline even when encryption of new objects is not required. The
encryption middleware is needed to handle GET requests for objects that may
have been previously encrypted. The keymaster is needed to provide keys for
those requests.

.. _container_sync_client_config:

Container sync configuration
----------------------------

If container sync is being used then the keymaster and encryption middlewares
must be added to the container sync internal client pipeline. The following
configuration steps are required:

#. Create a custom internal client configuration file for container sync (if
   one is not already in use) based on the sample file
   `internal-client.conf-sample`. For example, copy
   `internal-client.conf-sample` to `/etc/swift/container-sync-client.conf`.
#. Modify this file to include the middlewares in the pipeline in
   the same way as described above for the proxy server.
#. Modify the container-sync section of all container server config files to
   point to this internal client config file using the
   ``internal_client_conf_path`` option. For example::

     internal_client_conf_path = /etc/swift/container-sync-client.conf

.. note::

    The ``encryption_root_secret`` value is necessary to recover any encrypted
    data from the storage system, and therefore, it must be guarded against
    accidental loss. Its value (and consequently, the custom internal client
    configuration file) should not be stored on any disk that is in any
    account, container or object ring.

.. note::

    These container sync configuration steps will be necessary for container
    sync probe tests to pass if the encryption middlewares are included in the
    proxy pipeline of a test cluster.

--------------
Implementation
--------------

Encryption scheme
-----------------

Plaintext data is encrypted to ciphertext using the AES cipher with 256-bit
keys implemented by the python `cryptography package
<https://pypi.python.org/pypi/cryptography>`_. The cipher is used in counter
(CTR) mode so that any byte or range of bytes in the ciphertext may be
decrypted independently of any other bytes in the ciphertext. This enables very
simple handling of ranged GETs.

In general an item of unencrypted data, ``plaintext``, is transformed to an
item of encrypted data, ``ciphertext``::

  ciphertext = E(plaintext, k, iv)

where ``E`` is the encryption function, ``k`` is an encryption key and ``iv``
is a unique initialization vector (IV) chosen for each encryption context. For
example, the object body is one encryption context with a randomly chosen IV.
The IV is stored as metadata of the encrypted item so that it is available for
decryption::

  plaintext = D(ciphertext, k, iv)

where ``D`` is the decryption function.

The implementation of CTR mode follows `NIST SP800-38A
<http://csrc.nist.gov/publications/nistpubs/800-38a/sp800-38a.pdf>`_, and the
full IV passed to the encryption or decryption function serves as the initial
counter block.

In general any encrypted item has accompanying crypto-metadata that describes
the IV and the cipher algorithm used for the encryption::

  crypto_metadata = {"iv": <16 byte value>,
                     "cipher": "AES_CTR_256"}

This crypto-metadata is stored either with the ciphertext (for user
metadata and etags) or as a separate header (for object bodies).

Key management
--------------

A keymaster middleware is responsible for providing the keys required for each
encryption and decryption operation. Two keys are required when handling object
requests: a `container key` that is uniquely associated with the container path
and an `object key` that is uniquely associated with the object path.  These
keys are made available to the encryption middleware via a callback function
that the keymaster installs in the WSGI request environ.

The current keymaster implementation derives container and object keys from the
``encryption_root_secret`` in a deterministic way by constructing a SHA256
HMAC using the ``encryption_root_secret`` as a key and the container or object
path as a message, for example::

  object_key = HMAC(encryption_root_secret, "/a/c/o")

Other strategies for providing object and container keys may be employed by
future implementations of alternative keymaster middleware.

During each object PUT, a random key is generated to encrypt the object body.
This random key is then encrypted using the object key provided by the
keymaster. This makes it safe to store the encrypted random key alongside the
encrypted object data and metadata.

This process of `key wrapping` enables more efficient re-keying events when the
object key may need to be replaced and consequently any data encrypted using
that key must be re-encrypted. Key wrapping minimizes the amount of data
encrypted using those keys to just other randomly chosen keys which can be
re-wrapped efficiently without needing to re-encrypt the larger amounts of data
that were encrypted using the random keys.

.. note::

    Re-keying is not currently implemented. Key wrapping is implemented
    in anticipation of future re-keying operations.


Encryption middleware
---------------------

The encryption middleware is composed of an `encrypter` component and a
`decrypter` component.

Encrypter operation
^^^^^^^^^^^^^^^^^^^

Custom user metadata
++++++++++++++++++++

The encrypter encrypts each item of custom user metadata using the object key
provided by the keymaster and an IV that is randomly chosen for that metadata
item. The encrypted values are stored as :ref:`transient_sysmeta` with
associated crypto-metadata appended to the encrypted value. For example::

  X-Object-Meta-Private1: value1
  X-Object-Meta-Private2: value2

are transformed to::

  X-Object-Transient-Sysmeta-Crypto-Meta-Private1:
    E(value1, object_key, header_iv_1); swift_meta={"iv": header_iv_1,
                                                    "cipher": "AES_CTR_256"}
  X-Object-Transient-Sysmeta-Crypto-Meta-Private2:
    E(value2, object_key, header_iv_2); swift_meta={"iv": header_iv_2,
                                                    "cipher": "AES_CTR_256"}

The unencrypted custom user metadata headers are removed.

Object body
+++++++++++

Encryption of an object body is performed using a randomly chosen body key
and a randomly chosen IV::

  body_ciphertext = E(body_plaintext, body_key, body_iv)

The body_key is wrapped using the object key provided by the keymaster and a
randomly chosen IV::

  wrapped_body_key = E(body_key, object_key, body_key_iv)

The encrypter stores the associated crypto-metadata in a system metadata
header::

  X-Object-Sysmeta-Crypto-Body-Meta:
      {"iv": body_iv,
       "cipher": "AES_CTR_256",
       "body_key": {"key": wrapped_body_key,
                    "iv": body_key_iv}}

Note that in this case there is an extra item of crypto-metadata which stores
the wrapped body key and its IV.

Entity tag
++++++++++

While encrypting the object body the encrypter also calculates the ETag (md5
digest) of the plaintext body. This value is encrypted using the object key
provided by the keymaster and a randomly chosen IV, and saved as an item of
system metadata, with associated crypto-metadata appended to the encrypted
value::

  X-Object-Sysmeta-Crypto-Etag:
    E(md5(plaintext), object_key, etag_iv); swift_meta={"iv": etag_iv,
                                                        "cipher": "AES_CTR_256"}

The encrypter also forces an encrypted version of the plaintext ETag to be sent
with container updates by adding an update override header to the PUT request.
The associated crypto-metadata is appended to the encrypted ETag value of this
update override header::

  X-Object-Sysmeta-Container-Update-Override-Etag:
      E(md5(plaintext), container_key, override_etag_iv);
      meta={"iv": override_etag_iv, "cipher": "AES_CTR_256"}

The container key is used for this encryption so that the decrypter is able
to decrypt the ETags in container listings when handling a container request,
since object keys may not be available in that context.

Since the plaintext ETag value is only known once the encrypter has completed
processing the entire object body, the ``X-Object-Sysmeta-Crypto-Etag`` and
``X-Object-Sysmeta-Container-Update-Override-Etag`` headers are sent after the
encrypted object body using the proxy server's support for request footers.

.. _conditional_requests:

Conditional Requests
++++++++++++++++++++

In general, an object server evaluates conditional requests with
``If[-None]-Match`` headers by comparing values listed in an
``If[-None]-Match`` header against the ETag that is stored in the object
metadata. This is not possible when the ETag stored in object metadata has been
encrypted. The encrypter therefore calculates an HMAC using the object key and
the ETag while handling object PUT requests, and stores this under the metadata
key ``X-Object-Sysmeta-Crypto-Etag-Mac``::

  X-Object-Sysmeta-Crypto-Etag-Mac: HMAC(object_key, md5(plaintext))

Like other ETag-related metadata, this is sent after the encrypted object body
using the proxy server's support for request footers.

The encrypter similarly calculates an HMAC for each ETag value included in
``If[-None]-Match`` headers of conditional GET or HEAD requests, and appends
these to the ``If[-None]-Match`` header. The encrypter also sets the
``X-Backend-Etag-Is-At`` header to point to the previously stored
``X-Object-Sysmeta-Crypto-Etag-Mac`` metadata so that the object server
evaluates the conditional request by comparing the HMAC values included in the
``If[-None]-Match`` with the value stored under
``X-Object-Sysmeta-Crypto-Etag-Mac``. For example, given a conditional request
with header::

  If-Match: match_etag

the encrypter would transform the request headers to include::

  If-Match: match_etag,HMAC(object_key, match_etag)
  X-Backend-Etag-Is-At: X-Object-Sysmeta-Crypto-Etag-Mac

This enables the object server to perform an encrypted comparison to check
whether the ETags match, without leaking the ETag itself or leaking information
about the object body.

Decrypter operation
^^^^^^^^^^^^^^^^^^^

For each GET or HEAD request to an object, the decrypter inspects the response
for encrypted items (revealed by crypto-metadata headers), and if any are
discovered then it will:

#. Fetch the object and container keys from the keymaster via its callback
#. Decrypt the ``X-Object-Sysmeta-Crypto-Etag`` value
#. Decrypt the ``X-Object-Sysmeta-Container-Update-Override-Etag`` value
#. Decrypt metadata header values using the object key
#. Decrypt the wrapped body key found in ``X-Object-Sysmeta-Crypto-Body-Meta``
#. Decrypt the body using the body key

For each GET request to a container that would include ETags in its response
body, the decrypter will:

#. GET the response body with the container listing
#. Fetch the container key from the keymaster via its callback
#. Decrypt any encrypted ETag entries in the container listing using the
   container key


Impact on other Swift services and features
-------------------------------------------

Encryption has no impact on :ref:`versioned_writes` other than that any
previously unencrypted objects will be encrypted as they are copied to or from
the versions container. Keymaster and encryption middlewares should be placed
after ``versioned_writes`` in the proxy server pipeline, as described in
:ref:`encryption_deployment`.

`Container Sync` uses an internal client to GET objects that are to be sync'd.
This internal client must be configured to use the keymaster and encryption
middlewares as described :ref:`above <container_sync_client_config>`.

Encryption has no impact on the `object-auditor` service. Since the ETag
header saved with the object at rest is the md5 sum of the encrypted object
body then the auditor will verify that encrypted data is valid.

Encryption has no impact on the `object-expirer` service. ``X-Delete-At`` and
``X-Delete-After`` headers are not encrypted.

Encryption has no impact on the `object-replicator` and `object-reconstructor`
services. These services are unaware of the object or EC fragment data being
encrypted.

Encryption has no impact on the `container-reconciler` service. The
`container-reconciler` uses an internal client to move objects between
different policy rings. The destination object has the same URL as the source
object and the object is moved without re-encryption.


Considerations for developers
-----------------------------

Developers should be aware that keymaster and encryption middlewares rely on
the path of an object remaining unchanged. The included keymaster derives keys
for containers and objects based on their paths and the
``encryption_root_secret``. The keymaster does not rely on object metadata to
inform its generation of keys for GET and HEAD requests because when handling
:ref:`conditional_requests` it is required to provide the object key before any
metadata has been read from the object.

Developers should therefore give careful consideration to any new features that
would relocate object data and metadata within a Swift cluster by means that do
not cause the object data and metadata to pass through the encryption
middlewares in the proxy pipeline and be re-encrypted.

The crypto-metadata associated with each encrypted item does include some
`key_id` metadata that is provided by the keymaster and contains the path used
to derive keys. This `key_id` metadata is persisted in anticipation of future
scenarios when it may be necessary to decrypt an object that has been relocated
without re-encrypting, in which case the metadata could be used to derive the
keys that were used for encryption. However, this alone is not sufficient to
handle conditional requests and to decrypt container listings where objects
have been relocated, and further work will be required to solve those issues.
