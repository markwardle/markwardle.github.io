# How to properly set up a PHP application with composer and deploy it to Heroku - Part 1

## Introduction

PHP has come a long way as a language over the past several years.  The language has made significant improvements, the frameworks have also become more advanced and more elegant, and the entire ecosystem has developed wonderfully, especially around Composer, the de facto PHP package manager.

There are plenty of great frameworks for PHP including Laravel, Symfony2, and Slim.  However, with a framework comes a lot of fluff, that is, extraneous functionality and code, that you may not ever use or even look at.  If you're like me, you only want what you actually need to be in your project.

The great news is that you don't need a framework in order to create a project that has all the benefits of a framework.  Additionally, the PHP ecosystem has moved in a direction such that vendors are creating very well made, self-contained components for practically everything you need in your application.  That means, you can, essentially, construct your own framework with other developer's code!  The one thing that frameworks give you that you can't get otherwise is the code that glues all the pieces together.  For that you are on your own.  Fortunately, I have created this handy guide to teach you how to do just that.  As a bonus, this tutorial will teach you how to deploy your application to the Heroku cloud hosting platform.

Once you've completed this tutorial you will have a fully functioning (though rather simple) blog application up and running on the interweb tubes.  The code is all open source, so feel free to use it (and improve it!) if you would like.  The completed project can be found on [github](https://github.com/markwardle/php-blog-tutorial).

Let's get started!