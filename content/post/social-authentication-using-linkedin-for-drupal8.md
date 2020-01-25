---
title: Social authentication using LinkedIn for Drupal 8
date: 2017-09-04
hero: "/img/linkedin.jpg"
excerpt: you need to implement social authentication on a Drupal 8 website, https://www.drupal.org/project/social_auth_linkedin might help you out.
authors:
  - Adrian Gheorghe
---

If you need to implement social authentication on a Drupal 8 website, https://www.drupal.org/project/social_auth_linkedin might help you out.

The story behind this module is a simple one. I needed to implement LinkedIn authentication on a project i was working on. The website (developed using Drupal 8) is a Human Resources/CV aggregator solution, so it needed to allow users or potential candidates to login and import their profile/cv from various social networks. That being said, i used the Social API suite, Google and Facebook auth modules already existed, so i ended up writing the Social Auth LinkedIn module and a few event subscribers to save the user bio or cv data. More on that in a different post.

The Social Auth LinkedIn module is developed on top of the Social API Drupal 8 modules developed by gvso. and heavily inspired by the Social Auth Google and Social Auth Facebook modules. The library PHP-LinkedIn-SDK developed by ashwinks is used for querying the LinkedIn API.

In order to install this module you need to require the package using Composer in order to retrieve the [PHP-LinkedIn-SDK](https://github.com/ashwinks/PHP-LinkedIn-SDK) library.

```bash
composer require "drupal/social_auth_linkedin"
```

Social Auth LinkedIn allows users to register and login to your Drupal site with their LinkedIn account. It is based on Social Auth and Social API projects.

This module adds the path user/login/linkedin which redirects the user to LinkedIn for authentication.

After LinkedIn has returned the user to your site, the module compares the email address provided by LinkedIn. If your site already has an account with the same email address, user is logged in. If not, a new user account is created.

Login process can be initiated from the "LinkedIn" button in the Social Auth block. Alternatively, site builders can place (and theme) a link to user/login/linkedin wherever on the site.

## Instructions

The library PHP-LinkedIn-SDK developed by ashwinks is used for querying the LinkedIn API. The module should be installed using Composer, in order to retrieve the PHP-LinkedIn-SDK library. The module is also dependent of the Social API and Social Auth modules.

### Create LinkedIn Application

Go to https://www.linkedin.com/developer/apps and create a new application. After creating it, make sure both r_basicprofile and r_emailaddress permissions are checked.

The Client ID and Client Secret values will be required when configuring the module

Your Oauth 2.0 authorized redirect url will be
[your_base_url]/user/login/linkedin/callback

### Install Social Auth LinkedIn

Install module using composer to take care of dependencies

```bash
composer require "drupal/social_auth_linkedin"
```

Activate the Social Auth LinkedIn module and configure the Client ID and Client Secret acquired using the values acquired from LinkedIn.

The LinkedIn Auth button should now be appearing in the Social Auth Block