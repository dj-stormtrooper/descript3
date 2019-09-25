# `options`

```js
//  Создаем какой-то блок.
//  Вместо de.block должно быть что-то конкретное: de.http, de.array, ...
//
const block = de.block( {

    //  Описание блока, специфичное для каждого типа блока.
    block: ...,

    options: {
        //  Объект с опциями.
        //  Набор опций одинаковый для всех типов блоков.
        ...
    },

} );
```

## Кратко про все опции

```js
de.block( {

    block: ...,

    options: {
        //  Название блока, для логов.
        name: 'my_api.my_method',

        //  Зависимости между блоками.
        id: some_id,
        deps: [ some_id_1, some_id_2, ... ],

        //  Возможность вычислить новые параметры для блока.
        params: ...,

        //  Возможность сделать что-нибудь до запуска блока, после запуска блока
        //  или в случае ошибки выполнения блока.
        before: ...,
        after: ...,
        error: ...,

        //  Таймаут выполнения.
        timeout: 1000,

        //  Параметры кэширования.
        key: ...,
        maxage: ...,
        cache: ...,

        //  Флаг о том, что блок является обязательным.
        //  Ошибка в нем приводит к ошибке родительского блока (de.array или de.object).
        required: true,

        //  Логгер.
        logger: ...,
    },
} )
```

## `options.name`

Используется для логгирования [http-блоков](./http_block.md).

Приходит в `options.logger` (см. ниже) в поле `event.request_params.name`:

```js
const logger = function( event ) {

    const name = event.request_params.name;

    ...
}

const block = de.http( {
    block: ...,

    options: {
        name: 'my_api.my_method',
        logger: logger,
    },
} );
```


## `options.id` и `options.deps`

См. [Работа с зависимостями](./deps.md).


## `options.params`

Позволяет изменить переданные блоку сверху параметры:

```js
const orig_params = {
    foo: 42,
};

const block = de.block( {
    options: {
        //  Вот сюда в params придет orig_params.
        //
        params: ( { params } ) => {
            console.log( params, params === orig_params );
            //  { foo: 42 }, true

            return {
                ...params,
                bar: 24,
            };
        },

        //  Дальше во все остальные колбэки, где используются params
        //  будет приходить уже новые объект. Например, в before:
        //
        before: ( { params } ) => {
            console.log( params );
            //  { foo: 42, bar: 24 }
        },
    },
} );

//  Запускаем блок с какими-то параметрами:
//
const result = await de.run( block, {
    params: orig_params,
} );
```

### Аргументы `options.params`

```js
options: {
    params: ( { params, context, deps } ) => {
        return {
            ...
        };
    },
},

options: {
    params: {
        foo: ( { params, context, deps } ) => ...,

        ...
    },
},
```


### `options.params` — объект

Вместо функции `options.params` можно задать объектом:

```js
options: {
    params: {
        foo: null,
        bar: 42,
        quu: ( { params, context, deps } ) => params.quu + 1,
    },
},
```

Вычисленные параметры будут объектом, у которого есть только эти ключи (`foo`, `bar`, `quu`).
Все остальное будет отфильтровано. У того, что осталось, значения вычисляются следующим образом:

  * `foo: null` означает, что будет взято соответствующее значение из переданных в блок параметров (т.е. это `params.foo`).
  * `bar: 42` означает тоже самое, что и `null`, но если `params.bar` не определено, то будет взято значение `42` (т.е. `42` — это дефолтное значение).
  * `quu: ( { params } ) => params.quu + 1` — тут все понятно, явно вычисляем значение.

В виде функции это можно переписать так:

```js
options: {
    params: ( { params } ) => {
        return {
            foo: params.foo,
            bar: ( params.bar === undefined ) ? 42 : params.bar,
            quu: params.quu + 1,
        };
    },
},
```

Для каких-то сложных вычислений обычно используется функция, но иногда удобнее и нагляднее использовать объект.
Особенно, когда нужно просто отфильтровать входящие параметры и оставить только нужные, плюс задать дефолтные значения.

### Наследование

При наследовании блоков их `options.params` вызываются последовательно. Сперва у родителя, затем у потомка.
Причем потомку в `params` приходит уже результат работы `options.params` родителя:

```js
const parent = de.block( {
    options: {
        //  Сперва вызовется эта функция, в params придет orig_params.
        //
        params: ( { params } ) => {
            return {
                ...params,
                parent: true,
            };
        },
    },
} );

const child = parent( {
    options: {
        //  Затем уже вызовется эта функция и сюда в params придет результат
        //  работы первой функции. Т.е. `{ foo: 42, parent: true }`.
        //
        params: ( { params } ) => {

            //  И дальше везде в работе блока будет использоваться этот объект.
            //  Т.е. `{ foo: 42, parent: true, child: true }`.
            //
            return {
                ...params,
                child: true,
            };
        },
    },
} );

const orig_params = {
    foo: 42,
};
de.run( child, { params: orig_params } );
```

Если где-то в цепочке наследования в `options.params` будет не функция, а объект, то произойдет ровно то же самое.
Результирующий объект вычисляется по описанным выше правилам, на вход подается результат работы предыдущего `options.params`
или те параметры, с которыми был запущен блок.


## `options.before`

Это хук, который выполняется перед основным экшеном блока.
Если он кидает ошибку или же возвращает что-то отличное от `undefined`,
то экшен не выполнятся и дальше выполняется хук `options.error` или `options.after` соответственно.

Мы можем досрочно завершить выполнение блока:

  * Вернув сразу какой-то результат.
  * Вернуть ошибку.
  * Вернуть результат или ошибку асинхронно (т.е. вернуть промис).


### Завершить блок с результатом

```js
options: {
    before: ( { deps } ) => {
        if ( !deps[ id ].foo ) {
            return {
                foo: 42,
            };
        }
    },
},
```

### Завершить блок ошибкой

```js
options: {
    before: ( { params } ) => {
        if ( !params.foo ) {
            throw de.error( {
                id: 'INVALID_PARAMS',
            } );
        }
    },
},
```

### Вернуть результат асинхронно

```js
options: {
    before: ( { params } ) => {
        if ( !params.foo ) {
            return new Promise( ... );
        }
    },
},
```

Ждем, пока промис завершится. Дальше:

  * Если он резолвится не `undefined`, то это результат работы блока.
  * Если он режектится, то это ошибка.

Последствия такие же, как и в синхронном варианте.


### Завершить выполнение всего `de.run`

Предыдущий пример завершает выполнение блока, но иногда нужно завершить весь "запрос"
(по факту это не запрос, а запуск de.run). Произошла фатальная ошибка, нужен редирект и т.д.

```js
const block = de.object( {
    block: {

        ...

        foo: de.object( {

            block: de.block( {
                ...

                options: {
                    before: ( { params, cancel } ) => {
                        if ( !params.foo ) {
                            //  Тут мы не просто завершаем текущий блок,
                            //  но и вообще все блоки, запущенные в этом de.run.
                            //
                            cancel.cancel( de.error( {
                                id: 'FATAL_ERROR',
                            } ) );
                        }
                    }
                },

            } ),

        } ),

        ...

    },
} );

try {
    const result = await de.run( block, ... );

} catch ( e ) {
    console.log( e );
    //  FATAL_ERROR
}
```

### Аргументы `options.before`

```js
options: {
    before: ( { params, context, deps, cancel } ) => {

    },
},
```

### Наследование

В отличие от всех остальных хуков, здесь **обратный** порядок выполнения.
Сперва выполнятеся `before` у потомка, а затем уже у родителя.

Потому что `before` служит для того, чтобы досрочно завершить выполнение блока.
Если родительский `before` не вернет `undefined`, то выполнение блока завершится
и до `before` потомка дело не дойдет совсем. И чтобы перекрыть это поведение,
`before` у потомка должен вызываться до.

```js
const parent = de.block( {
    options: {
        //  ВНИМАНИЕ! Тут обратный порядок, в сравнении со всеми остальными хуками.
        //  Сперва вызывается before а потомка, а потом уже у родителя.
        //
        //  Если он вернет что-то или бросит ошибку, то экшен блока вызываться не будет.
        //
        before: ( { params } ) => {
    
        },
    },
} );

const child = parent( {
    options: {
        //  Сперва вызовется этот before.
        //  Если он вернет что-то или бросит ошибку,
        //  то выполнение блока завершится, before у родителя вызвано не будет.
        //
        params: ( { params } ) => {
            return {
                ...params,
                child: true,
            };
        },
    },
} );

de.run( child, ... );
```


## `options.after`

Тут все как в `options.before`, только наоборот.
Если экшен блока завершился успешно, вызывается хук `after`.

При помощи `after` можно:

  * Изменить возвращаемое блоком значение.
  * Передумать и вернуть таки ошибку.
  * Сделать все это же, но асинхронно (вернув промис).
  * Выполнить какие-то дополнительные действия (выставить куки, например).

### Вернуть другое значение

```js
options: {
    after: ( { result } ) => {
        //  Модифицируем результат.
        //
        return {
            ...result,
            foo: 42,
        };
    },
},
```

### Вернуть ошибку

```js
options: {
    after: ( { result } ) => {
        if ( !result.foo ) {
            //  Что-то пошло не так, результат нас не устраивает.
            //  Кидаем ошибку.
            //
            throw de.error( {
                id: 'INVALID_RESULT',
            } );
        }
    },
},
```

### Асинхронный after

Тут так же, как и в `options.before`. Можно вернуть промис, дождемся его результатов, дальше как в синхронном варианте.

### Выставить куки

```js
options: {
    after: ( { result, context } ) => {
        if ( result.foo ) {
            const { res } = context;

            //  Выставляем в ответ куки (например, сессионные и т.д.)
            //
            res.cookie( 'foo', result.foo, ... );
        }
    },
},
```

### Аргументы `options.after`

```js
options: {
    after: ( { params, context, deps, cancel, result } ) => {

    },
},
```

### Наследование

Здесь стандартный порядок выполнения хуков: сперва родительский `after`, затем `after` потомка.

```js
const parent = de.block( {
    options: {
        //  Если экшен блока успешно завершился, то выполнится родительский after.
        //  И в result придет результат экшена.
        //
        after: ( { result } ) => {
            return {
                //  Возвращаем новый результат.
                //
                ...result,
                foo: 42,
            }
    
        },
    },
} );

const child = parent( {
    options: {
        //  Если родительский after отработал без ошибок и вернул что-то (не undefined),
        //  то это что-то приходит в after потомка в качестве результата.
        //
        after: ( { result } ) => {
            return {
                ...result,
                bar: 24,
            };
        },
    },
} );

de.run( child, ... );
```

Т.е. цепочка `after` (которая может быть длиннее двух элементов) работает начиная со старейшего предка, ему на вход попадает результат экшена.
Каждый следующий элемент получает на вход результат работы предыдущего. И результат последнего уже становится финальным результатом работы блока.
Если где-то в этой цепочке происходит ошибка (`throw` или любая другая), то весь этот процесс сразу завершается и эта ошибка отправляется в `options.error`.


## `options.error`

Это как `options.after`, то опять наоборот.
Если где-то в процессе выполнения блока произошла ошибка, то мы всегда попадаем в `options.error`.
Здесь мы можем:

  * Поменять ошибку на другую.
  * Передумать и вместо ошибки вернуть результат (не ошибку).
  * То же самое, но асинхронно.


### Вернуть другую ошибку

```js
options: {
    error: ( { error } ) => {
        //  Какая-то неправильная ошибка.
        //  Возвращаем другую взамен.
        //
        if ( error.error.id === 'SOME_ERROR' ) {
            throw de.error( {
                id: 'ANOTHER_ERROR',
            } );
        }

    },
},
```

### Вернуть вместо ошибки результат

```js
options: {
    error: ( { error } ) => {
        if ( error.error.id === 'SOME_ERROR' ) {
            //  На самом деле это только так кажется, что это ошибка!
            //  Возвращаем результат.
            //
            return {
                foo: 42,
            };
        }
    },
},
```

### Асинхронный `options.error`

Можем вернуть промис, дальше как обычно.

### Аргументы `options.error`

```js
options: {
    error: ( { params, context, deps, cancel, error } ) => {

    },
},
```

### Наследование

Цепочка из `options.error` работает так же, как и `options.error`.
Но. Тут наоборот, если предыдущий элемент кидает ошибку, то она приходит в аргументе в поле `error`
в следующий элемент цепочки. А если возвращает не `undefined`, то это считается результатом работы блока
и, как обычно, он попадает в `options.after`.


## Ошибка `de.ERROR_ID.TOO_MANY_AFTERS_OR_ERRORS`

```js
options: {
    after: () => { throw Error(); },
    error: () => null,
}
```

Вот этот код потенциально мог бы зациклиться.
Как бы не выполнялся блок, мы в конце концов попадем либо в `after`, либо в `error`.
Из `error` мы всегда возвращаем не `undefined`, значит это новый результат работы блока и он уходит в `after`.
А из `after` мы всегда кидаем ошибку и значит попадаем обратно в `error`.

Чтобы зацикливания не случалось, `descript` считает количество итераций в цепочке `error -> after -> error -> ...`.
И если их больше N (в данный момент 3), то блок окончательно и бесповоротно заканчивается с ошибкой с id `de.ERROR_ID.TOO_MANY_AFTERS_OR_ERRORS`.


## `options.timeout`

Позволяет задать таймаут блока в миллисекундах.
Если блок работает дольше, то он завершает свою работу с ошибкой `de.ERROR_ID.BLOCK_TIMED_OUT`.

```js
options: {
    //  Таймаут 1 секунда.
    //
    timeout: 1000,
},
```

Два важных момента:

  * Выполнение блока считается со всеми его фазами.
    С ожиданием зависимостей, выполнением `before`, `after`, собственно экшена, выполнением подблоков и т.д.

  * `options.timeout` это не то же самое, что `http_block.timeout` — этот таймаут только про http-запросы и наоборот,
    не включает в себя ничего, кроме времени работы http-запроса.


## `options.key`, `options.maxage`, `options.cache`

Эти три параметра позволяют закэшировать блок и при последующем запуске брать результат из кэша.

Для начала нужно задать кэш, в который мы будем писать и из которого будем читать данные.
Кэш это объект с двумя методами:

```js
const cache = {
    get: function( key ) { ... },
    set: function( key, value, maxage ) { ... },
};
```

  * Метод `get` должен вернуть данные, соответствующие переданному ключу. Синхронно или в виде промиса.
  * Метод `set` соответственно, должен сохранить значение `value` по ключу `key` на период `maxage`.
    Результат работы `set` игнорируется.

Есть встроенный очень простой кэш `de.Cache`:

```js
const cache = new de.Cache();
```

Он хранит все просто в памяти, `maxage` задается в миллисекундах. Нет ограничений на количество записей.
Так что для каких-то серьезных ситуаций лучше его не использовать, а использовать [descript2-memcached](https://github.com/doochik/descript2-memcached).

Параметры `key` и `maxage` задают ключ и срок хранения:

```js
options: {
    //  Кэшируем по имени метода и параметру id.
    key: ( { params } ) => `my_method:id=${ params.id }`,
    //  Храним 1 час.
    maxage: 60 * 60 * 1000,
    //  Задаем в каком кэше все храним.
    cache: cache,
},
```


## `options.required`

Применимо только к подблокам [de.array](./array_block.md) и [de.object](./object_block.md).

```js
const block = de.object( {
    block: {

        foo: block_foo,

        bar: block_bar( {
            options: {
                required: true,
            },
        } ),

    },
} );
```

Если в процессе выполнения этого блока `block_foo` завершится ошибкой, а `block_bar` нет,
то результатом работы `block` будет что-то типа:

```js
{
    foo: {
        error: {
            id: 'SOME_ERROR',
        },
    },
    bar: {
        //  Some result.
        bar: 24,
    },
}
```

Т.е. несмотря на ошибку в `block_foo`, сам блок `block` завершится удачно.

Если же теперь наоборот, `block_foo` завершается успешно, а `block_bar` ошибкой.
В этом случае, блок `block` тоже завершится вот такой ошибкой:

```js
{
    error: {
        id: de.ERROR_ID.REQUIRED_BLOCK_FAILED,
        //  Эта строчка показывает нам, кто именно из подблоков завершился ошибкой.
        path: '.bar',
        //  Это та самая ошибка в required-блоке, из-за которой весь de.object упал с ошибкой.
        reason: {
            error: {
                id: 'SOME_ERROR',
            },
        },
    },
}
```

Таким образом, чтобы `de.object` или `de.array` успешно завершились, необходимо (но, очевидно, не достаточно),
чтобы все required-блоки завершились успешно.


## `options.logger`

Позволяет логировать некоторые события.
В данный момент, это события связанные с http-запросами.
Возможно, в будущем появится что-то еще, поэтому оно в `options`,
а не в `http_block.logger`.

Что такое логгер. Это объект с одним методом `log`, в который приходят события:

```js
const de = require( 'descript' );

const logger = {
    log: function( event, context ) {
        switch ( event.type ) {

            case de.Logger.EVENT.REQUEST_START: {
                const { request_options } = event;
                ...
                break;
            }

            case de.Logger.EVENT.REQUEST_SUCCESS: {
                const { request_options, result, timestamps } = event;
                ...
                break;
            }

            case de.Logger.EVENT.REQUEST_ERROR: {
                const { request_options, error, timestamps } = event;
                ...
                break;
            }

        }
    },
};

const block = de.block( {
    options: {
        logger: logger,
    },
} );
```

В поле `event.request_options` содержится много всего, описывающего http-запрос, который мы логируем.
В частности, в `event.request_options.http_options` содержится объект,
который был отправлен в нодовский метод `http.request` (или `https.request`).