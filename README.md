# Good software development practices

[![Run in Cisco Cloud IDE](https://static.production.devnetcloud.com/codeexchange/assets/images/devnet-runable-icon.svg)](https://developer.cisco.com/codeexchange/devenv/NSO-developer/nso-service-dev-practices)

## Loopback Python example

Open the `~/src/nso-service-dev-practices/loopback/python/loopback/loopback.py` example again and study its contents.

```python
# -*- mode: python; python-indent: 4 -*-
import ipaddress
import ncs
from ncs.application import Service


class ServiceCallbacks(Service):
    @Service.create
    def cb_create(self, tctx, root, service, proplist):
        self.log.info('Service create(service=', service._path, ')') # pylint: disable=protected-access

        management_prefix = service.management_prefix
        self.log.debug(f'Value of management-prefix leaf is {management_prefix}')
        net = ipaddress.IPv4Network(management_prefix)
        management_address = list(net.hosts())[0]

        bgp_prefix = service.bgp_prefix
        self.log.debug(f'Value of bgp-prefix leaf is {bgp_prefix}')
        net = ipaddress.IPv4Network(bgp_prefix)
        bgp_address = list(net.hosts())[0]
        tvars = ncs.template.Variables()
        tvars.add('MANAGEMENT_ADDRESS', management_address)
        tvars.add('BGP_ADDRESS', bgp_address)
        template = ncs.template.Template(service)
        template.apply('loopback-template', tvars)
```

First thing that we can improve in the given code is to use a function that will calculate the management IP address and bgp address from the given prefix. With this we avoid repeating the same code for the calculation multiple times and we can also use it in the future.

