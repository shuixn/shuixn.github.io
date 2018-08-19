---
title: 【funcipher】PHP密文定制工具
date: 2018-02-04
categories:
  - 技术
tags: 
  - PHP 
  - composer
---

funcipher
======
Custom random ciphertext

# Install

```bash
composer require "funsoul/funcipher: 2.0"
```

# Usage

Global Variable

```php
CIPHER_USE_LOWER
CIPHER_USE_CAPITAL
CIPHER_USE_NUMBER
CIPHER_USE_SPECIAL
```
### create()
```php
use Funsoul\Funcipher\Funcipher;

$cipher = new Funcipher();
echo $cipher->create(10);
// ,+T=!V67|E

```
### ignore()
```php
use Funsoul\Funcipher\Funcipher;

$cipher = new Funcipher();
echo $cipher->ignore(['a',1,3,5,7])->create(10);
// w0/22S0i2~
```
### Customize code
```php
use Funsoul\Funcipher\Funcipher;

$cipher = new Funcipher();
echo $cipher->ignore([1,3,5,7])->create(10,[CIPHER_USE_NUMBER]);
// 4288062890
```

# License

MIT