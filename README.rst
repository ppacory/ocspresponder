OCSP Responder
==============

.. image:: https://img.shields.io/travis/threema-ch/ocspresponder/master.svg?maxAge=2592000
    :target: https://travis-ci.org/threema-ch/ocspresponder
    :alt: Build status

.. image:: https://img.shields.io/pypi/v/nine.svg?maxAge=2592000
    :target: https://pypi.python.org/pypi/ocspresponder
    :alt: PyPI Version

.. image:: https://img.shields.io/pypi/dm/ocspresponder.svg?maxAge=2592000
    :target: https://pypi.python.org/pypi/ocspresponder
    :alt: PyPI Downloads

.. image:: https://img.shields.io/pypi/l/ocspresponder.svg?maxAge=2592000
    :target: https://pypi.python.org/pypi/ocspresponder
    :alt: License

.. image:: https://img.shields.io/pypi/pyversions/ocspresponder.svg?maxAge=2592000
    :target: https://pypi.python.org/pypi/ocspresponder
    :alt: Python Versions

.. image:: https://img.shields.io/pypi/status/ocspresponder.svg?maxAge=2592000
    :target: https://pypi.python.org/pypi/ocspresponder
    :alt: Stability Status

RFC 6960 compliant OCSP Responder framework written in Python 3.5+.

It is based on the ocspbuilder_ and asn1crypto_ libraries. The HTTP
server is implemented using Bottle_.

Current status: Alpha. Don't use for production yet.


Features
--------

**Goals**

- Simple: Don't overwhelm the user with a gazillion options.
- Flexible: Configurable using Python code.

**Supported extensions**

- Nonce (RFC 6960 Section 4.4.1)

**Not (yet) implemented**

- Multiple certificates per request / response


Usage
-----

Right now, ``ocspresponder`` assumes the usage of a custom keypair just for
signing OCSP responses.

To be able to instantiate the ``OCSPResponder`` server, you need to provide
this keypair as well as the certificate of the issueing CA.

.. code-block:: python

   ISSUER_CERT = 'path/to/issuer_cert.pem'
   OCSP_CERT = 'path/to/responder_cert.pem'
   OCSP_KEY = 'path/to/responder_key.pem'

Furthermore you need to provide two custom functions:

- A function that – given a certificate serial – will return the appropriate
  ``CertificateStatus`` and - depending on the status - a revocation
  datetime.
- A function that – given a certificate serial – will return the corresponding
  certificate as a string.

You're expected to implement these functions yourself. In the future, drop-in
libraries for typical use cases could be provided.

Example implementations:

.. code-block:: python

   from ocspresponder import OCSPResponder, CertificateStatus
   
   def validate(serial: int) -> (CertificateStatus, Optional[datetime]):
       if certificate_is_valid(serial):
           return (CertificateStatus.good, None)
       elif certificate_is_expired(serial):
           expired_at = get_expiration_date(serial)
           return (CertificateStatus.revoked, expired_at)
       elif certificate_is_revoked(serial):
           revoked_at = get_revocation_date(serial)
           return (CertificateStatus.revoked, revoked_at)
       else:
           return (CertificateStatus.unknown, None)
   
   def get_cert(serial: int) -> str:
       """
       Assume the certificates are stored in the ``certs`` directory with the
       serial as base filename.
       """
       with open('certs/%s.cert.pem' % serial, 'r') as f:
           return f.read().strip()

If an exception occurs in either of the two functions, an OCSP response with
the ``response_status`` set to ``internal_error`` will be returned.

Now you can instantiate the ``OCSPResponder`` and launch the development server:

.. code-block:: python

   print('Initializing OCSP responder')
   app = OCSPResponder(
       ISSUER_CERT, OCSP_CERT, OCSP_KEY,
       validate_func=validate,
       cert_retrieve_func=get_cert,
   )
   print('Starting OCSP responder')
   app.serve(port=8080, debug=True)


Type hints
----------

This library uses the optional type hints as defined in PEP484_. The ``typing``
module is only provided in Python 3.5+, but older versions of Python could run
the code as well if the ``typing`` module is installed from PyPI.


Testing
-------

To run the test, install ``requirements-dev.txt`` using pip and run pytest::

    py.test -v


Release process
---------------

Update version number in ``setup.py`` and ``CHANGELOG.md``::

    vim -p setup.py CHANGELOG.md

Do a commit and signed tag of the release::

    export VERSION={VERSION}
    git add setup.py CHANGELOG.md
    git commit -m "Release v${VERSION}"
    git tag -u C75D77C8 -m "Release v${VERSION}" v${VERSION}

Build source and binary distributions::

    python3 setup.py sdist
    python3 setup.py bdist_wheel

Sign files::

    gpg --detach-sign -u C75D77C8 -a dist/ocspresponder-${VERSION}.tar.gz
    gpg --detach-sign -u C75D77C8 -a dist/ocspresponder-${VERSION}-py3-none-any.whl

Register package on PyPI::

    twine3 register -r pypi-threema dist/ocspresponder-${VERSION}.tar.gz

Upload package::

    twine3 upload -r pypi-threema dist/ocspresponder-${VERSION}*
    git push
    git push --tags


License
-------

::

    Copyright 2016 Threema GmbH

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.


.. _ocspbuilder: https://github.com/wbond/ocspbuilder
.. _asn1crypto: https://github.com/wbond/asn1crypto
.. _Bottle: http://bottlepy.org/docs/dev/index.html
.. _PEP484: https://www.python.org/dev/peps/pep-0484/
