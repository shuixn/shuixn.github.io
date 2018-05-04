title: call_user_func在php7和php5表现不一样
date: 2018-05-04
tags: 
  - PHP 
  - PHP源码
---

之前，在这篇文章{% post_link PHP返回对象的公有属性和值 %}留下了一个疑问，

```php
trait PrintPublic {
    public function publics()
    {
        return call_user_func('get_object_vars', $this);
    }
}

class User {
    private $age = 18;
    public $name = 'xxx';

    use PrintPublic;
}

$user = new User();
$item = $user->publics();
print_r($item);

// version < php 7
// Array
// (
//     [name] => xxx
// )

// version > php 7
// Array
// (
//     [name] => xxx
//     [age] => 18
// )
```

先找到 **call_user_func** 源码，

```bash
cd your-php-src/ext/standard/basic_functions.c
```

### php7.1

```c
/* {{{ proto mixed call_user_func(mixed function_name [, mixed parmeter] [, mixed ...])
   Call a user function which is the first parameter
   Warning: This function is special-cased by zend_compile.c and so is usually bypassed */
PHP_FUNCTION(call_user_func)
{
	zval retval;
	zend_fcall_info fci;
	zend_fcall_info_cache fci_cache;

	ZEND_PARSE_PARAMETERS_START(1, -1)
		Z_PARAM_FUNC(fci, fci_cache)
		Z_PARAM_VARIADIC('*', fci.params, fci.param_count)
	ZEND_PARSE_PARAMETERS_END();

	fci.retval = &retval;

	if (zend_call_function(&fci, &fci_cache) == SUCCESS && Z_TYPE(retval) != IS_UNDEF) {
		if (Z_ISREF(retval)) {
			zend_unwrap_reference(&retval);
		}
		ZVAL_COPY_VALUE(return_value, &retval);
	}
}
/* }}} */
```

### php5.6

```c
/* {{{ proto mixed call_user_func(mixed function_name [, mixed parmeter] [, mixed ...])
   Call a user function which is the first parameter */
PHP_FUNCTION(call_user_func)
{
	zval *retval_ptr = NULL;
	zend_fcall_info fci;
	zend_fcall_info_cache fci_cache;

	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "f*", &fci, &fci_cache, &fci.params, &fci.param_count) == FAILURE) {
		return;
	}

	fci.retval_ptr_ptr = &retval_ptr;

	if (zend_call_function(&fci, &fci_cache TSRMLS_CC) == SUCCESS && fci.retval_ptr_ptr && *fci.retval_ptr_ptr) {
		COPY_PZVAL_TO_ZVAL(*return_value, *fci.retval_ptr_ptr);
	}

	if (fci.params) {
		efree(fci.params);
	}
}
/* }}} */
```

待续..