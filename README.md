rpc_exceptions
=========

Simple, transparent, name-spaced exceptions across JSON-RPC services.

Version
----

0.1

Installation
--------------

```
pip install rpc_exceptions
```

Examples
--------

Define a base exception with an api_name, exceptions for that API, and instantiate RPCExceptionHandler with those exceptions.

```
from rpc_exceptions import RPCExceptionHandler, WrappedRPCError

class TestError(WrappedRPCError):
    api_name = 'test'

class PEBCAKError(TestError):
    code = -1
    _default_message = 'Problem exists between chair and keyboard'

class FUBARError(TestError):
    code = -2
    _default_message = 'FUBARed'

error_handler = RPCExceptionHandler([ PEBCAKError, FUBARError ])
```

Decorate exposed functions with the instantiated RPCExceptionHandler's wrap_rpc_exception decorator.

```
from wrapped_rpc.exceptions import error_handler, PEBCAKError, FUBARError
@error_handler.wrap_rpc_exception
def fail1(arg):
    raise PEBCAKError

@error_handler.wrap_rpc_exception
def fail2(arg):
    raise FUBARError

```

Coerce error codes and messages to exceptions  the instantiated RPCExceptionHandler's get_exception_instance method.

```
#client using tinyrpc
from wrapped_rpc.exceptions import error_handler, PEBCAKError, FUBARError
from tinyrpc import RPCClient, RPCError
from tinyrpc.protocols.jsonrpc import JSONRPCProtocol
from tinyrpc.transports.http import HttpPostClientTransport

class WrappedRPCClient(RPCClient):
    def __init__(self, protocol, transport, api_name):
        self.protocol = protocol
        self.transport = transport
        self.api_name = api_name

    def _send_and_handle_reply(self, req):
        reply = self.transport.send_message(req.serialize())
        response = self.protocol.parse_reply(reply)
        if hasattr(response, 'error'):
            try:
                raise error_handler.get_exception_instance(
                         response._jsonrpc_error_code,
                         self.api_name, response.error)
            except KeyError:
                raise RPCError('Error calling remote procedure: %s' %\
                               response.error)
        return response

rpc_client = WrappedRPCClient(
    JSONRPCProtocol(),
    HttpPostClientTransport('http://localhost:1234'),
    'test'
)
server = rpc_client.get_proxy()

try:
    server.fail1('foo')
except PEBCAKError as e:
    print 'code: %s, message: %s' % (e.code, e.msg)

try:
    server.fail2('bar')
except FUBARError as e:
    print 'code: %s, message: %s' % (e.code, e.msg)

```

License
----

3-BSD
