# Feature toggle management inside the codebase

The following example is based on [this](https://martinfowler.com/articles/feature-toggles.html) article.

## Context

Let's suppose we are creating a module that will store sales orders in the database. We will have the following domain objetcs:


```python
import typing
```


```python
class Product(typing.NamedTuple):
    id: int
    price: float
```


```python
class SalesOrderItem(typing.NamedTuple):
    id: int
    product_id: int
    quantity: int
```


```python
class SalesOrder(typing.NamedTuple):
    id: int
    client_id: int
    items: typing.List[SalesOrderItem]
```

Now, in:order to save these object in the DB we have implemented the following repositories:


```python
class SalesOrderRepository:
    def save(self, order: SalesOrder):
        print('Storing the sales order: ', order.id)
```


```python
class SalesOrderItemRepository:
    def save_many(self, items: typing.List[SalesOrderItem]):
        print('Storing multiple sales order items: ', [item.id for item in items])
```

When it is time to save a sales order we call the following function


```python
def create_sales_order(order: SalesOrder):
    order_repository = SalesOrderRepository()
    items_repository = SalesOrderItemRepository()
    
    order_repository.save(order)
    items_repository.save_many(order.items)
```


```python
order = SalesOrder(
    id=1,
    client_id=1,
    items=[
        SalesOrderItem(id=1, product_id=1, quantity=10),
        SalesOrderItem(id=2, product_id=15, quantity=25),
    ]
)

create_sales_order(order)
```

    Storing the sales order:  1
    Storing multiple sales order items:  [1, 2]


So far so good. Now a new business requirement comes in. We need to send a creation email after the sales order creation. We decide to introduce a feature toggle, the create_sales_order function will now be like this:


```python
def get_feature_toggles():
    return {
        'new_awsome_feature': True,
    }


def send_email(order: SalesOrder):
    print('sending email to client', order)
```


```python
def create_sales_order(order: SalesOrder):
    order_repository = SalesOrderRepository()
    items_repository = SalesOrderItemRepository()
    
    order_repository.save(order)
    items_repository.save_many(order.items)
    
    if get_feature_toggles()['new_awsome_feature']:
        send_email(order)
```


```python
order = SalesOrder(
    id=1,
    client_id=1,
    items=[
        SalesOrderItem(id=1, product_id=1, quantity=10),
        SalesOrderItem(id=2, product_id=15, quantity=25),
    ]
)

create_sales_order(order)
```

    Storing the sales order:  1
    Storing multiple sales order items:  [1, 2]
    sending email to client SalesOrder(id=1, client_id=1, items=[SalesOrderItem(id=1, product_id=1, quantity=10), SalesOrderItem(id=2, product_id=15, quantity=25)])


That's ok but, why does the create_sales_order need to know about the **new_awsome** feature? We can change that by using an abstraction layer


```python
class FeatureDecisions:
    def __init__(self):
        _toggles = get_feature_toggles()
        self.sales_order_creations_should_send_email = _toggles['new_awsome_feature']

def get_feature_decisions():
    return FeatureDecisions()
```


```python
def create_sales_order(order: SalesOrder):
    order_repository = SalesOrderRepository()
    items_repository = SalesOrderItemRepository()
    
    order_repository.save(order)
    items_repository.save_many(order.items)
    
    feature_decisions = get_feature_decisions()
    
    if feature_decisions.sales_order_creations_should_send_email:
        send_email(order)
```


```python
order = SalesOrder(
    id=1,
    client_id=1,
    items=[
        SalesOrderItem(id=1, product_id=1, quantity=10),
        SalesOrderItem(id=2, product_id=15, quantity=25),
    ]
)

create_sales_order(order)
```

    Storing the sales order:  1
    Storing multiple sales order items:  [1, 2]
    sending email to client SalesOrder(id=1, client_id=1, items=[SalesOrderItem(id=1, product_id=1, quantity=10), SalesOrderItem(id=2, product_id=15, quantity=25)])


Now the **create_sales_order** function is agnostic of the **new_awsome_feature**, that is fine, but we can do better. We will pass the **feature_decisions** to invert the dependency of this object so testing is easier.


```python
def create_sales_order(order: SalesOrder, feature_decisions: FeatureDecisions):
    order_repository = SalesOrderRepository()
    items_repository = SalesOrderItemRepository()
    
    order_repository.save(order)
    items_repository.save_many(order.items)
    
    if feature_decisions.sales_order_creations_should_send_email:
        send_email(order)
```


```python
order = SalesOrder(
    id=1,
    client_id=1,
    items=[
        SalesOrderItem(id=1, product_id=1, quantity=10),
        SalesOrderItem(id=2, product_id=15, quantity=25),
    ]
)


create_sales_order(order, get_feature_decisions())
```

    Storing the sales order:  1
    Storing multiple sales order items:  [1, 2]
    sending email to client SalesOrder(id=1, client_id=1, items=[SalesOrderItem(id=1, product_id=1, quantity=10), SalesOrderItem(id=2, product_id=15, quantity=25)])


Cool! But there is one problem. If we need to change the codebase in multiple places we would end up having multiple if statements. To fix this problem we can use the factory an strategy patterns


```python
def _current_create_sales_order(order: SalesOrder):
    order_repository = SalesOrderRepository()
    items_repository = SalesOrderItemRepository()
    
    order_repository.save(order)
    items_repository.save_many(order.items)


def _create_sales_order_and_send_email(order: SalesOrder):
    _current_create_sales_order(order)
    send_email(order)

        
def get_sales_order_creator():
    feature_decisions = get_feature_decisions()
    
    if feature_decisions.sales_order_creations_should_send_email:
        return _create_sales_order_and_send_email
    
    return _current_create_sales_order
```


```python
order = SalesOrder(
    id=1,
    client_id=1,
    items=[
        SalesOrderItem(id=1, product_id=1, quantity=10),
        SalesOrderItem(id=2, product_id=15, quantity=25),
    ]
)

create_sales_order = get_sales_order_creator()
create_sales_order(order)
```

    Storing the sales order:  1
    Storing multiple sales order items:  [1, 2]
    sending email to client SalesOrder(id=1, client_id=1, items=[SalesOrderItem(id=1, product_id=1, quantity=10), SalesOrderItem(id=2, product_id=15, quantity=25)])


That is much better, now all the if statements are encapsulated in one single function: **get_sales_order_creator**

Once the **awesome_feature** implementation is done we can remove the feature toggles, we can remove the **get_sales_order_creator** function and call directly the new function


```python
def create_sales_order(order: SalesOrder):
    order_repository = SalesOrderRepository()
    items_repository = SalesOrderItemRepository()
    
    order_repository.save(order)
    items_repository.save_many(order.items)

    send_email(order)
```


```python
create_sales_order(order)
```

    Storing the sales order:  1
    Storing multiple sales order items:  [1, 2]
    sending email to client SalesOrder(id=1, client_id=1, items=[SalesOrderItem(id=1, product_id=1, quantity=10), SalesOrderItem(id=2, product_id=15, quantity=25)])

