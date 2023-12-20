```python
class ServiceCallbacks(Service):
    def cb_create(self, tctx, root, service, proplist):
        bgp_prefix = service.bgp_prefix

        self.log.debug(f'Value of bgp-prefix leaf is {bgp_prefix}')
```
