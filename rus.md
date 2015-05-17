# Составление синхронных и асинхронных функций в JavaScript

Наше приложение для примера реализует функцию `createEmployee`, которая создает
сотрудника по его персональному ID.

Для создания сотрудника, наша система загружает некоторые данные из базы,
проверяет эти данные, и затем выполняет вставку. Некоторые из наших функций
написаны в стиле передачи продолжений (то есть, принимают коллбэк-функцию *([
рограммирование в стиле передачи продолжений][1] - прим.пер.)*), а некоторые в
прямом стиле (то есть, возвращают значения и/или пробрасывают исключения). Нам
хотелось бы составить  данные функции таким образом, чтобы они обрабатывали
успешное выполнение и неудачное как единое целое – чтобы любая ошибка, на любом
шаге последовательности вызывала последующие шаги, которые иначе были бы
пропущены. Но когда часть проверок выполняется синхронно, а часть - асинхронно,
такое организовать непросто.

Чтобы обойти эту проблему, программисты встраивают анонимные функции, передающие
возвращаемые значения и исключения в коллбэки. Например:

    function hasWildHair(person) {
        return person.hairColor !== 'Зеленые' || person.hairColor !== 'Розовые'
    }
    
    function isOfAge(person) {
        return person.age > 17;
    }
    
    function ensureDriversLicense(person, callback) {
        Database.getDriversLicenseByPersonId(person.id, function(err, license) {
          if (err) {
              callback(err);
          }
          else if (license.expired) {
              callback('Человек должен иметь действующие водительские права.');
          }
          else {
              callback(null, person);
          }
        });
    }
    
    function createEmployee(personId, callback) {
        var workflow = async.compose(
            Database.createEmployee,
            ensureDriversLicense,
            ensureWashingtonAddress,
            function(person, callback) {
                if (hasWildHair(person)) {
                    callback(null, person);   
                }
                else {
                    callback('Человек должен иметь дикие волосы.');
                }
            },
            function(person, callback) {
                if (isOfAge(person)) {
                    callback(null, person);
                }
                else {
                    callback('Человеку должно быть больше 18 лет.');
                }
            }
            Database.getPersonById);
        
        workflow(personId, callback);
    }
    

При таком подходе возникает несколько проблем:

1.  Мы включили две одноразовые функции, которые добавляют визуальный шум и
некоторые затраты на обслуживание.
   
2.  Наш код в прямом стиле (проверка цвета волос и возраста) недоступен за
пределами оборачивающих его функций.
   
3.  Наши одноразовые функции содержат логику преобразования функций-предикатов,
возвращающих false, в сообщения об ошибке. Нужно проверить человека на наличие
диких волос десять раз в коде? Десять раз добавьте проверку на `null` и вызов
сообщения «Человек должен иметь дикие волосы.»

В этом посте я продемонстрирую способ использовать функции высшего порядка,
переносящие функции в прямом стиле в мир коллбэков – включая композицию без
введения обычных функциональных выражений. После этого, код выше будет
выглядеть так:

    var createEmployee = async.compose(
        Database.createEmployee,
        ensureDriversLicense,
        ensureWashingtonAddress,
        asyncify(ensureWildHair),
        asyncify(ensureOfAge),
        Database.getPersonById);
    

### Шаг 1: Пишем функции, пробрасывающие ошибки

Для начала, мы напишем функцию высшего порядка, которая будет принимать предикат,
в значение которого будет передаваться предикат, и ошибку для пробрасывания в
случае, если наше значение не соответствует предикату. Зачем нужно пробрасывать
ошибку? Это  убирает различия от того, выполнилась ли функция успешно или нет.

    function ensure(predicate, error, value) {
        if (predicate(value) {
            return value;
        }
        else {
            throw error;
        }
    }
    

Теперь мы можем составить `ensure` из наших предикатов, создав переиспользуемые
валидаторы, пробрасывающие ошибки:

    var ensureWildHair = _.partial(ensure, hasWildHair, 'Человек должен иметь дикие волосы.');
    var ensureOfAge    = _.partial(ensure, ofAge, 'Человеку должно быть больше 18 лет.');
    

…что переместит часть обработки ошибок за пределы наших больших функций создания сотрудника:

    function createEmployee(personId, callback) {
        var workflow = async.compose(
            Database.createEmployee,
            ensureDriversLicense,
            ensureWashingtonAddress,
            function(person, callback) {
                try {
                    callback(null, ensureHasWildHair(person));
                }
                catch (e) {
                    callback(e);
                }
            },
            function(person, callback) {
                try {
                    callback(null, ensureOfAge(person));
                }
                catch (e) {
                    callback(e);
                }
            }
            Database.getPersonById);
        
        workflow(personId, callback);
    }
    

### Шаг 2: Работаем в мире коллбэков

Мы избавились от последствий применения одноразовых функций и собрали создание
ошибок и обработку предикатов внутри обобщающей функции-утилиты `ensure`.
Теперь, мы напишем некоторый код, который позволит нам использовать функции в
прямом стиле в контексте стиля передачи продолжений.

    function asyncify(f) {
      return function _asyncify() {
        var args = Array.prototype.slice.call(arguments, 0);
        var argsWithoutLast = args.slice(0, -1);
        var callback = args[args.length-1];
        var result, error;
    
        try {
          result = f.apply(this, argsWithoutLast);
        }
        catch (e) {
          error = e;
        }
    
        setTimeout(function() {
          callback(error, result);
        }, 0);
      }
    }
    


Эта новая функция позволяет нам отобразить любую n-ную функцию, которая либо
возвращает  значение, либо пробрасывает ошибку к (n+1)-ой функции, последний
аргумент которой ожидается коллбэком в стиле Node.js, первым аргументом которого
станет пробрасываемая ошибка (если такая существует), и вторым аргументом будет
возвращаемое значение (если мы ничего не пробрасываем).

Пример:

    function head(xs) {
        if (_.isArray(xs) && !_.isEmpty(xs)) {
            return xs[0];
        }
        else {
            throw "Can't get head of empty array.";
        }
    }
    
    var cbHead = asyncify(head);
    
    cbHead([1, 2, 3], function(err, result) {
        // result === 1
    });
    
    cbHead([], function(err, result) {
        // err === "Can't get head of empty array.";
    });
    

Теперь мы можем использовать функцию `asyncify` для превращения наших функций в
прямом стиле в функции, принимающие коллбэк.

    function createEmployee(personId, callback) {
        var workflow = async.compose(
            Database.createEmployee,
            ensureDriversLicense,
            ensureWashingtonAddress,
            asyncify(ensureHasWildHair),
            asyncify(ensureOfAge),
            Database.getPersonById);
        
        workflow(personId, callback);
    }
    

Наконец, мы можем исключить функциональное выражение полностью; вместо этого мы
определим `createEmployee` как объединение остальных функций. Так как выражение
не делает ничего более, чем делегирование функции с теми же самыми
характеристиками, мы можем спокойно убрать его.

    var createEmployee = async.compose(
        Database.createEmployee,
        ensureDriversLicense,
        ensureWashingtonAddress,        
        asyncify(ensureHasWildHair), 
        asyncify(ensureOfAge),
        Database.getPersonById);
    

### Заключение

Наша финальная имплементация сокращает количество специального кода,
подключаемого для валидации и создания сотрудника, до абсолютного минимума.
Получившееся приложение является очень модульным; `ensure` и `asyncify` могут
быть использованы в различных контекстах за пределами `createEmployee`. Наконец,
мы обобщили вещи до такой степени,  что нашей работой стало лишь составление
обобщенных функций вместе для  создания чего-то специфического под возникающие
задачи.

### Почитать по теме (что писал не я)

1. [“Достаточно хорошая” обработка ошибок Clojure][2]
2. [Промисы — монады асинхронного программирования][3]
3. [Ошибки монад в Clojure][4]


### Почитать по теме (что написано мной)

1. [Уборка приложения на JavaScript с помощью функций высшего порядка][5]
2. [Чумовой функционал на каррированном JavaScript][6]



[1]: https://ru.wikipedia.org/wiki/%D0%9F%D1%80%D0%BE%D0%B4%D0%BE%D0%BB%D0%B6%D0%B5%D0%BD%D0%B8%D0%B5_(%D0%B8%D0%BD%D1%84%D0%BE%D1%80%D0%BC%D0%B0%D1%82%D0%B8%D0%BA%D0%B0)#.D0.9F.D1.80.D0.BE.D0.B3.D1.80.D0.B0.D0.BC.D0.BC.D0.B8.D1.80.D0.BE.D0.B2.D0.B0.D0.BD.D0.B8.D0.B5_.D0.B2_.D1.81.D1.82.D0.B8.D0.BB.D0.B5_.D0.BF.D0.B5.D1.80.D0.B5.D0.B4.D0.B0.D1.87.D0.B8_.D0.BF.D1.80.D0.BE.D0.B4.D0.BE.D0.BB.D0.B6.D0.B5.D0.BD.D0.B8.D0.B9
[2]: http://adambard.com/blog/acceptable-error-handling-in-clojure/
[3]: https://blog.jcoglan.com/2011/03/11/promises-are-the-monad-of-asynchronous-programming/
[4]: https://brehaut.net/blog/2011/error_monads
[5]: http://blog.carbonfive.com/2015/01/05/tidying-up-a-javascript-application-with-higher-order-functions/
[6]: http://blog.carbonfive.com/2015/01/14/gettin-freaky-functional-wcurried-javascript/
