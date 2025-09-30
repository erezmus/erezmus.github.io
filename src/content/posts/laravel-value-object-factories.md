---
title: 'Laravel value object factories'
published: 2025-09-30
draft: false
description: 'How to leverage laravel factories for value object classes'
tags: ['PHP', 'Laravel']
---

## Introduction

If you are not familiar with value objects, they are immutable objects that store values. They do not
have any functionality nor does a value object have an identiy field.

When developing tests or seeders for your Laravel applications, it's useful to be able to generate fake
data for value objects as well as models.

Let's take the example of a contact value object:

`app/ValueObjects/Contact.php`

```php
<?php

namespace App\ValueObjects;

use Database\Factories\ContactFactory;
use Illuminate\Database\Eloquent\Attributes\UseFactory;
use Illuminate\Database\Eloquent\Factories\HasFactory;

#[UseFactory(ContactFactory::class)]
class Contact
{
    use HasFactory;

    public function __construct(public string $name, public string $email) {}
}

```

Our factory can then be:

`database/factories/ContactFactory.php`

```php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;

class ContactFactory extends Factory
{
    public function definition()
    {
        return [
            'name' => fake()->name,
            'email' => fake()->email,
        ];
    }
}

```

Let's use it in a test:

`tests/Unit/ContactTest.php`

```php
<?php

use App\ValueObjects\Contact;

describe('contact', function () {
    test('make a contact', function () {
        Contact::factory()->make();

        expect($contact->name)->not->toBe(null);
        expect($contact->email)->not->toBe(null);
    });
});


```

Running the test, we get this error:

```sh
/app # php artisan test tests/Unit/ContactTest.php

   FAIL  Tests\Unit\ContactTest
  ⨯ contact → make a contact                                                                                                                                 0.52s
  ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   FAILED  Tests\Unit\ContactTest > `contact` → make a contact                                                                                          TypeError
  App\ValueObjects\Contact::__construct(): Argument #1 ($name) must be of type string, array given, called in /app/vendor/laravel/framework/src/Illuminate/Database/Eloquent/Factories/Factory.php on line 851
```

The problem is that Eloquent factories instantiate the target object by passing an associative array when instantiating the model:

`vendor/laravel/framework/src/Illuminate/Database/Eloquent/Factories/Factory.php`

```php
//...
public function newModel(array $attributes = [])
{
    $model = $this->modelName();

    return new $model($attributes);
}
//...
```

## Single instance

We need to override the `newModel` method. Let's modify our factory to override.

`database/factories/ContactFactory.php`

```php
class ContactFactory extends Factory
{
    //... rest of ContactFactory

    public function newModel(array $attributes = [])
    {
        $model = $this->modelName();

        return new $model($attributes['name'], $attributes['email']);
    }
}

```

Re running the test now works:

```sh
/app # php artisan test tests/Unit/ContactTest.php -vvv

  PASS  Tests\Unit\ContactTest
  ✓ contact → make a contact                                                                                                                                 0.66s

  Tests:    1 passed (2 assertions)
  Duration: 0.73s
```

## Collection

What if we want multiple Contact instances in our tests:

`tests/Unit/ContactsTest.php`

```php
//...
    test('make multiple contacts', function () {
        $contacts = Contact::factory()->count(2)->make();

        expect($contacts[0]->name)->not->toBe(null);
        expect($contacts[0]->email)->not->toBe(null);
    });
//...
```

We now have another issue:

```sh
/app # php artisan test tests/Unit/ContactTest.php -vvv

   FAIL  Tests\Unit\ContactTest
  ✓ contact → make a contact                                                                                                                                 0.53s
  ⨯ contact → make multiple contacts                                                                                                                         0.07s
  ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   FAILED  Tests\Unit\ContactTest > `contact` → make multiple contacts                                                                             ErrorException
  Undefined array key "name"

  at vendor/laravel/framework/src/Illuminate/Foundation/Bootstrap/HandleExceptions.php:258
    254▕      */
    255▕     protected function forwardsTo($method)
    256▕     {
    257▕         return fn (...$arguments) => static::$app
  ➜ 258▕             ? $this->{$method}(...$arguments)
    259▕             : false;
    260▕     }
    261▕
    262▕     /**

  1   vendor/laravel/framework/src/Illuminate/Foundation/Bootstrap/HandleExceptions.php:258
  2   database/factories/ContactFactory.php:21
  3   vendor/laravel/framework/src/Illuminate/Database/Eloquent/Factories/Factory.php:435
  4   tests/Unit/ContactTest.php:14


  Tests:    1 failed, 1 passed (2 assertions)
  Duration: 0.67s
```

Now the issue is that Laravel's `Factory` base class instantiates a model without any attributes when creating multiple instances:

```php
            //...
            $instances = $this->newModel()->newCollection(array_map(function () use ($parent) {
                return $this->makeInstance($parent);
            }, range(1, $this->count)));
            //...
```

The solution is to override the `make` method as well:

`database/factories/ContactFactory.php`

```php
    // ...
    public function make($attributes = [], ?\Illuminate\Database\Eloquent\Model $parent = null)
    {
        if (! empty($attributes)) {
            return $this->state($attributes)->make([], $parent);
        }

        if ($this->count === null) {
            return $this->newModel($this->getExpandedAttributes($parent));
        }

        if ($this->count < 1) {
            return new \Illuminate\Support\Collection;
        }

        return new \Illuminate\Support\Collection(array_map(function () use ($parent) {
            return $this->newModel($this->getExpandedAttributes($parent));
        }, range(1, $this->count)));
    }
```

This is a simplified version of the base Factory method that preservers the necessary parts of the parent method that is
tailored for value objects.

````

The test now works properly:

```sh
/app # php artisan test tests/Unit/ContactTest.php -vvv

   PASS  Tests\Unit\ContactTest
  ✓ contact → make a contact                                                                                                                       0.95s
  ✓ contact → make multiple contacts                                                                                                               0.12s

  Tests:    2 passed (5 assertions)
  Duration: 1.19s

````

## Refactoring

We can now create a base class to keep the common functionality and we can overrided the `create` method to simply call `make`. Value Objects are not stored in the database so we can avoid other errors by disabling the saving to the database.

`database/factories/ValueObjectFactory.php`

```php
<?php


namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Collection;

abstract class ValueObjectFactory extends Factory
{
    /**
     * Create a collection of value objects.
     *
     * @param  (callable(array<string, mixed>): array<string, mixed>)|array<string, mixed>  $attributes
     * @return \Illuminate\Support\Collection<int, TRelatedModel>|TRelatedModel
     */
    public function make($attributes = [], ?Model $parent = null)
    {
        if (! empty($attributes)) {
            return $this->state($attributes)->make([], $parent);
        }

        if ($this->count === null) {
            return tap($this->makeInstance($parent), function ($instance) {
                $this->callAfterMaking(new Collection([$instance]));
            });
        }

        if ($this->count < 1) {
            return new Collection;
        }

        return new Collection(array_map(function () use ($parent) {
            return $this->newModel($this->getExpandedAttributes($parent));
        }, range(1, $this->count)));
    }

    /**
     * Create a collection of models and persist them to the database.
     *
     * @param  (callable(array<string, mixed>): array<string, mixed>)|array<string, mixed>  $attributes
     * @param  \Illuminate\Database\Eloquent\Model|null  $parent
     * @return \Illuminate\Database\Eloquent\Collection<int, TModel>|TModel
     */
    public function create($attributes = [], ?Model $parent = null)
    {
        return $this->make($attributes, $parent);
    }
}
```

And we update our factory accordingly.

`database/factories/ContactFactory.php`

```php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;

class ContactFactory extends ValueObjectFactory
{
    public function definition()
    {
        return [
            'name' => fake()->name,
            'email' => fake()->email,
        ];
    }

    public function newModel(array $attributes = [])
    {
        $model = $this->modelName();

        return new $model($attributes['name'], $attributes['email']);
    }
}
```
