---
created: 2026-09-12T23:09
updated: 2026-09-12T23:22
---


## Intuition
- goal: reusable & easy-inherited classes 
	- beneficial to collaborative development, where projects are divided into groups
	- reusability, scalability and efficiency
- terms
	- method: function inside the classes
	- attribute: variable like 
	- `self`: pass the object itself as the first argument (always passed unless it's `@staticmethod`)
## Magic methods (dunder)
- def: special methods to enable custom object behavior (strong presets)
- always surrounded by double underscores (`__xxx__`)
### constructor `__init__`
- <span style="color:rgb(255, 0, 0)">auto-run once when class is created</span>
- argument type
	- positional arg: must defined by user
	- keyword argument: have default value
```python
# coder
class Item:
	def __init__(self, name: str, price: float, quantity=0):
		# validation
		assert price >= 0, "whatever error messages"
		
		# define attributes
		self.name=name
		self.price=price
		self.quantity=quantity

# user
item=Item("phone", 100)
```

### Other common magic methods



