# Function Type vs Method Type

``` python
class _C:
    def _m(self): pass


print(f"Method type: {type(_C()._m)}")
```

```
<class 'method'>
```

``` python
def _f():
    pass

print(f"Function type: {type(_f)}")
```

```
Function type: <class 'function'>
```
