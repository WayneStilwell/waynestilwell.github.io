---
layout: post
title: "Laravel Migration Errors"
date: 2019-08-27 13:45:20 -0600
description: Errors I encountered running migrations with Laravel 5.8 and MySQL 5.6.
img: # Add image post (optional)
tags: [PHP,Laravel,MySQL]
---

I started a new Udemy course where we build a real time SPA forum using Laravel and VueJS. It looks pretty cool!

One of the first things we do is setup the back-end. This invovled creating some models and migrations in Laravel, using `php artisan make:model`.

The models are really basic, so I figured this would take 5 minutes. I was wrong. 

# First Error
My first attempt to run `php artisan migrate` gave the following error:

> #1071 - Specified key was too long; max key length is 767 bytes

I use my personal laptop for work (my work computer is OLD). Because of that, I have to run MySQL 5.6 locally. It turns out that in 5.6, the max length of an index is 767 bytes (as shown in the error above). In MySQL 5.7, that limit was increased to 3072 bytes.

## Obvious Solution
Update MySQL to at least 5.7. 

However, I can't because I need 5.6 for work. I could look into running multiple MySQL versions (sounds like the perfect use case for Docker!), but I don't have time for that right now.

## The Other Piece of the Problem

Consider this Laravel migration code:

```php
$table->string('email')->unique();
```

This adds a new email column of type `VARCHAR(255)`, and creates a unique index on it.

The problem is that Laravel uses the __utf8mb4__ character set (to store emojis). This uses up to 4 bytes per character. A string with 255 characters using 4 bytes per character gives you 1,020 bytes. Since 1,020 > 767, this is why I'm getting the "key too long" error.

It sounds like what I really want to do is change the default length of `string()` in Laravel. Turns out I can!

__app/Providers/AppServiceProvider.php:__
```php
use Illuminate\Support\Facades\Schema;

public function boot()
{
    Schema::defaultStringLength(191);
}
```

I added this and re-ran the migration. But I now got a new error...

# Error Number Two

> Error 1215: Cannot add foreign key constraint

Two of the tables we created were __questions__ and __replies__. The code that the instructor gave to generate these are below.

__questions__ Table Migration:
```php
public function up()
{
    Schema::create('questions', function (Blueprint $table) {
        $table->increments('id');
        $table->string('title');
        $table->string('slug');
        $table->text('body');
        $table->integer('category_id')->unsigned();
        $table->integer('user_id')->unsigned();
        $table->timestamps();
    });
}
```

__replies__ Table Migration:
```php
public function up()
{
    Schema::create('replies', function (Blueprint $table) {
        $table->increments('id');
        $table->text('body');
        $table->integer('question_id')->unsigned();
        $table->integer('user_id');
        $table->foreign('question_id')->references('id')->on('questions')->onDelete('cascade');
        $table->timestamps();
    });
}
```

I would not have encountered this SQL error if I had copied his code verbatim. However, he had us generate the migration files using `php artisan make:model XYZ -mfr'. This auto-generated the `id` and timestamp fields for us.

Notice in the instructor's code above how the `id` fields are created using the `increments()` method?

It turns out that in Laravel 5.8, they changed the default for auto-incremented IDs to use `bigIncrements()`. In short, my version of the migration created `questions.id` as an unsigned BIGINT, as opposed to an unsigned INT the instructor's code created.

## Solution
Everywhere I reference an ID in a foreign table, I replaced `integer` with `bigInteger`. So my migrations for those two tables looked like this.

__questions__ Table Migration:
```php
public function up()
{
    Schema::create('questions', function (Blueprint $table) {
        $table->bigIncrements('id');
        $table->string('title');
        $table->string('slug');
        $table->text('body');
        $table->bigInteger('category_id')->unsigned();
        $table->bigInteger('user_id')->unsigned();
        $table->timestamps();
    });
}
```

__replies__ Table Migration:
```php
public function up()
{
    Schema::create('replies', function (Blueprint $table) {
        $table->bigIncrements('id');
        $table->text('body');
        $table->bigInteger('question_id')->unsigned();
        $table->bigInteger('user_id')->unsigned();
        $table->foreign('question_id')
            ->references('id')
            ->on('questions')
            ->onDelete('cascade');
        $table->timestamps();
    });
}
```

Boom!
