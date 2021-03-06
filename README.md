# DBCart  


Shopping Cart library for Laravel > 5.5 and Laravel 6 that uses database instead of sessions to store carts.



## Features

* Cart for guest users
* Cart for logged in users
* Guest Cart is merged with User Cart when logged in
* Singleton Cart instance to avoid unnecessary database queries. But also possible to avoid signleton cart if needed.
* Built on top of Eloquent Model, so easily extendable and all eloquent methods can be used.
* Multiple instances of cart
* Schedule expired carts for deletion

## Before installation
if you already use <a href="https://github.com/hassansin/dbcart" target="_blank">`dbcart by Hassansin`</a>, please skip DB migrations
and publishing configuration. This package is fully compatible. 
## Installation
1. Install the package via composer:
    ``` bash
    composer require nabeelalalmai/dbcart-laravel
    ```

    The package will automatically register itself.

2. Publish database migrations(IF YOU USE dbcart:

    ```bash
    php artisan vendor:publish --provider="NabeelAlalmai\DBcart\CartServiceProvider" --tag="migrations"
    ```

    After the migration has been published you can create the `cart` and `cart_lines` tables by running the migrations:

    ```bash
    php artisan migrate
    ```
   
   

## Configuration

Optionally, you can publish package config file:

```sh
php artisan vendor:publish --provider="NabeelAlalmai\DBcart\CartServiceProvider" --tag=config
```
Now, update `config/cart.php` if required 

## Usage

#### Get Cart Instance:
Get the current cart instance. It returns a singleton cart instance:
```php
$cart = app('cart'); //using app() helper
```
or,

```php
$cart = App::make('cart');
```
alternatively, you can avoid singleton instance and use the model class to load the cart instance from database everytime:

```php
use NabeelAlalmai\DBcart\Models\Cart;
//...
$cart = Cart::current();
```
The idea of using singleton cart is that the cart object will be available globally throughout your app (e.g. controllers/models/views/view composers etc) for a single request. Also as you manipulate cart items, `$cart->item_count` and `$cart->total_price` would get updated.

#### Add an Item: `$cart->addItem($attributes)`

```php
$cart->addItem([
    'product_id' => 1,
    'unit_price' => 10.5,
    'quantity' => 1
]); 
```

which is equivalent to `$cart->items()->create($attributes)`

#### Get Items: `$cart->items`

Since `$cart` is eloquent model instance, you can use any of the eloquent methods to get items

```php
$items = $cart->items // by dynamic property access
$items = $cart->items()->get()  
$items = $cart->items()->where('quantity', '>=', 2)->get()
```

#### Update an Item: `$cart->updateItem($where, $attributes)`

```php  
$cart->updateItem([
    'id' => 2
], [
    'product_id' => 1,
    'unit_price' => 10.5,
    'quantity' => 1
]); 
```

which is equivalent to `$cart->items()->where($where)->first()->update($attributes)`

#### Remove an Item: `$cart->removeItem($where)`

```php
$cart->removeItem([
    'id' => 2
]); 
```
    
which is equivalent to `$cart->items()->where($where)->first()->delete()`

#### Clear Cart Items: `$cart->clear()`

Remove all items from the cart

```php
$cart->clear(); 
```    
#### Checkout cart: `$cart->checkout()`

This method only updates `status` and `placed_at` column values. `status` is set to `pending`

```php    
$cart->checkout();

```


#### Move item(s) between carts:

To move all items from one cart to another cart instance:

```php
$cart = app('cart');
$wishlist = app('cart', ['name' => 'wishlist']);

//move all wishlist items to cart
$wishlist->moveItemsTo($cart);
```

To move a single item between carts:

```php
$cart = app('cart');
$wishlist = app('cart', ['name' => 'wishlist']);

//move an wishlist item to cart
$item = $wishlist->items()->where(['product_id' => 1])->first();
$item->moveTo($cart);

```


#### Get Cart Attributes

```php
$total_price = $cart->total_price; // cart total price
$item_count = $cart->item_count; // cart items count
$date_placed = $cart->placed_at; // returns Carbon instance
```
#### Cart Statuses

Supports several cart statuses:
* `active`: currently adding items to the cart
* `expired`: cart is expired, meaningful for session carts
* `pending`: checked out carts 
* `completed`: completed carts 

```php
use NabeelAlalmai\DBcart\Models\Cart;

// get carts based on their status: active/expired/pending/complete
$active_carts = Cart::active()->get();
$expired_carts = Cart::expired()->get();
$pending_carts = Cart::pending()->get();
$completed_carts = Cart::completed()->get();
```
#### Working with Multiple Cart Instances

By default, cart instances are named as `default`. You can load other instances by providing a name:

```php
$cart = app('cart'); // default cart, same as: app('cart', [ 'name' => 'default'];
$sales_cart = app('cart', [ 'name' => 'sales']);
$wishlist = app('cart', [ 'name' => 'wishlist']);
```
or, without singleton carts:

```php
use NabeelAlalmai\DBcart\Models\Cart;
//...
$cart = Cart::current();
$sales_cart =  Cart::current('sales');
$wishlist =  Cart::current('wishlist');
```

To get carts other than `default`:

```php
$pending_sales_carts = Cart::instance('sales')->pending()->get();
```

#### Delete Expired Carts:

The guest carts depend on the session lifetime. They are valid as long as the session is not expired. You can increase session lifetime in `config/session.php` to increase cart lifetime. When a session expires, a new cart instance will be created in database and the old one will no longer be used. Over time, these expired carts could pile-up in database. 

Laravel Task Scheduler comes to the rescue. To enable scheduler just add following to the crontab in your server:

    * * * * * php /path/to/artisan schedule:run >> /dev/null 2>&1 

That's it. The module will now check for expired carts in every hour and delete them. This won't affect the carts for loggedin users.

#### Other features:

Get Cart User: `$cart->user`

Get Item Product: `$item->product`

Is Cart Empty: `$cart->isEmpty()`

If an item exists in cart: `$cart->hasItem(['id' => 10])`

Expire the cart: `cart->expire();`

Set to `completed` status: `$cart->complete();`


## Extending Cart Model
It's easy to extend DBCart. You can extend base DBCart model and add your own methods or columns. Follow these steps to extend the cart model:

1. Create a model by extending `NabeelAlalmai\DBcart\Models\Cart`:
    ```php    
    namespace App;
        
    use NabeelAlalmai\DBcart\Models\Cart as BaseCart;

    class Cart extends BaseCart
    {
        //override or add your methods here ...

        public function getSubTotalAttribute(){
            return $this->attributes['total_price'];
        }
        public function getGrandTotalAttribute(){
            //taking discount, tax etc. into account            
            return $this->sub_total - $this->discount;
        }
        
    }
    ```
2. Update `cart_model` in `config/cart.php` with the fully qualified class name of the extended model.
  
    ```php
    'cart_model' => App\Cart::class,
    ```
3. That's it, you can now load the cart as usual:

    ```php
    $cart = App::make('cart');
    ```

You can also follow the above steps and create your own `CartLine` model by extending `NabeelAlalmai\DBcart\Models\CartLine`. Be sure to update `config/cart.php` to reflect your changes.

## Disclaimer
I was using <a href="https://github.com/hassansin/dbcart" target="_blank">`dbcart by Hassansin`</a> but it doesn't support Laravel 6. So, I re-wrote the package to support Laravel > 5.5. If you use dbcart by Hassansin you can use this package and it should work with no conflicts.  

## Support

Reach out to me at one of the following places!

- Twitter at <a href="http://twitter.com/nabeel_alalmai" target="_blank">`Nabeel Al Almai - نبيل الالمعي`</a>

---
