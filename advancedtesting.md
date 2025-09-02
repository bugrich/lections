# JS: Продвинутое тестирование

# Table of contents

## Введение

Основные темы:

- Тестирование ошибок. Снепшот-тесты.
- Фикстуры. Организация тестовых данных.
- Изоляция побочных эффектов. Стабы. Инверсия зависимости.
- Моки. Тестирование методом черного ящика.
- Таймеры. Управление временем.
- Тестирование асинхронного кода.

[книга по тестам](https://github.com/goldbergyoni/javascript-testing-best-practices)

---

## Тестирование ошибок

Основные тесты, которые нужно писать, это тесты на успешные сценарии работы. Но в некоторых ситуациях код должен возвращать ошибки и их тоже бывает нужно проверять. Под ошибками понимаются ситуации, в которых код выбрасывает исключение. В чем их особенность? Посмотрите на тест:

```JS
test('boom!', () => {
  try {
    functionWithException(0)
  }
  catch (e) {
    expect(e).not.toBeNull()
  }
})
```

Этот код пытается протестировать ситуацию, при которой функция `functionWithException()` выбрасывает исключение, если ей передать 0. Как вы думаете, этот тест проверит, что функция действительно порождает исключение?

Правильный ответ — нет. Если функция `functionWithException()` не выбросит исключение, то тест пройдет, так как код не попадет в блок `catch`.

Документация Jest предлагает свой способ тестирования таких ситуаций. Jest позволяет указать количество утверждений, которые должны выполниться в тесте. Если этого не происходит, то Jest сообщает об ошибке:

```js
test('boom!', () => {
  // Количество утверждений, которые должны быть запущены в этом тесте
  expect.assertions(1)

  try {
    functionWithException(0)
  } catch (e) {
    expect(e).not.toBeNull()
  }
})
```

Этот способ крайне опасен. Он порождает хрупкие тесты, которые завязаны на то, как они написаны. Если вы захотите добавить новое утверждение, то тест провалится и придется его править. Вам всегда придется следить за тем, чтобы это число было правильным. Не используйте этот подход, чем больше контекстной зависимости, тем сложнее разобраться в коде и проще наделать ошибок.

И наконец-то мы подобрались к правильному способу. В Jest есть матчер, который самостоятельно отлавливает исключение и проверяет, что оно вообще было сгенерировано.

```js
test('boom!', () => {
  expect(() => {
    functionWithException(0)
  }).toThrow()
})
```

Главная особенность этого матчера в том, что он принимает на вход функцию, которая вызывается внутри. Благодаря этому, он может самостоятельно отследить появление исключения. Этот код не содержит неявного состояния и лишних проверок, он делает ровно то, что нужно делать и не требует от нас слишком много. Более того, теоретически возможен тест, в котором делается сразу несколько проверок на различные исключения. Это значительно сложнее провернуть с предыдущими способами.

Иногда важно не просто поймать исключение, но и убедиться в том, что это ожидаемое исключение. Сделать это можно, передав в матчер `toThrow()` строку, которая должна присутствовать в сообщении исключения.

```js
test('boom!', () => {
  expect(() => {
    functionWithException(0)
  }).toThrow('divide by zero')
})
```

### Tests

#### My

```js
test('boom!', () => {
  expect(() => {
    read('/undefined')
  }).toThrow('no such file')
})

test('boom1!', () => {
  expect(() => {
    read('./__tests__')
  }).toThrow('directory')
})
```

#### Teacher

```js
test('read', () => {
  expect(() => read('/undefined')).toThrow()
  expect(() => read('/etc')).toThrow()
})
```

---

## Побочные эффекты

Проще всего тестировать код, состоящий из чистых функций. Данные на вход, результат на выходе. Никаких неожиданностей, никакого состояния, никакого взаимодействия с внешним миром.

```js
import _ from 'lodash'

expect(_.last([1, 2, 3])).toBe(3)
```

Далеко не весь код можно назвать чистым. Без побочных эффектов не обходится ни одна реальная программа. Результаты вычислений нужно куда-то записать, отправить, сохранить.

Побочные эффекты резко усложняют тестирование, требуют более глубоких навыков написания тестов и лучшего понимания того, как организовывать такой код.

Вот лишь некоторые примеры использования побочных эффектов:

- Случайные числа
- HTTP-запросы
- Отправка писем
- Взаимодействие с базой данных
- Взаимодействие с глобальными переменными
- Чтение и запись файлов
- Оперирование текущим временем

В чем заключается сложность? Представьте себе функцию, которая выполняет отправку письма пользователю:

```js
if (sendGreetingEmail(user)) {
  // Вывести на сайте сообщение о том, что письмо было отправлено
}
```

Вот ее гипотетический тест:

```js
expect(sendGreetingEmail(user)).toBe(true)
```

С этим тестом определенно не все в порядке. Все, что мы тут проверяем — то, что функция возвращает `true`, но мы ничего не знаем о том, отправляет ли эта функция письмо и если отправляет, то какое? Все ли нормально с этим письмом?

Но реальность еще сложнее. Отправлять настоящие письма ни в коем случае нельзя. Во-первых, это не этично по отношению к людям. Во-вторых, даже если отправлять письма на фейковые аккаунты, мы все равно взаимодействуем с внешней системой. Внешние системы — это долго, такие тесты будут выполняться значительно дольше по времени, чем тесты чистых функций. Кроме того, любое взаимодействие с внешней средой не детерминировано. Начиная от того, что сеть не надежна, и эти тесты могут падать с ошибками без видимой на то причины, заканчивая тем, что, за слишком частую отправку писем почтовая служба может заблокировать IP-адрес с которого идет отправка. И, наконец, это может быть небезопасно.

И это только отправка писем. С другими побочными эффектами будут другие сложности. И для их решения потребуются другие подходы к организации кода и тестов. В течение нескольких следующих уроков, мы разберем наиболее распространенные примеры побочных эффектов и того, как правильно с ними работать.

### Tests

#### My

```js
import getFunction from '../functions.js'

const buildUser = getFunction()

// BEGIN (write your solution here)
test('main feature', () => {
  const expectedProperties = ['email', 'firstName', 'lastName']
  const user = buildUser()

  expectedProperties.forEach((key) => expect(user).toHaveProperty(key))

  let user1 = buildUser()
  expect(user1).not.toEqual(user)

  const customUser = buildUser({ firstName: 'Petya' })
  expect(customUser).toHaveProperty('firstName', 'Petya')
})
```

#### Teacher

```js
import getFunction from '../functions.js'

const buildUser = getFunction()

// BEGIN
test('buildUser fields', () => {
  const user1 = buildUser()
  expect(user1).toEqual(
    expect.objectContaining({
      email: expect.any(String),
      firstName: expect.any(String),
      lastName: expect.any(String),
    })
  )
})

test('buildUser random', () => {
  const user1 = buildUser()
  const user2 = buildUser()
  expect(user1).not.toEqual(user2)
})

test('buildUser override', () => {
  const newData1 = {
    email: 'test@email.com',
  }
  const user1 = buildUser(newData1)
  expect(user1).toMatchObject(newData1)
})
```

---

## Тестирование кода, взаимодействующего с файлами

Наиболее типичный побочный эффект — взаимодействие с файлами (файловые операции). В основном это либо чтение файлов, либо запись в них. С чтением разбираться значительно проще, поэтому с него и начнем.

### Чтение файлов

В большинстве случаев чтение файлов не доставляет особых проблем. Оно ничего не изменяет и выполняется локально, в отличие от сетевых запросов. Это значит, что мы вряд ли столкнемся со случайными ошибками, если у нас есть необходимый файл и нужные права.

При тестировании функций, читающих файлы, должно выполняться ровно одно условие. Функция должна позволять менять путь до файла. В таком случае, достаточно создать файл нужной структуры в фикстурах.

```js
// Функция читает файл со списком пользователей системы и возвращает их имена
// В линуксе это файл /etc/passwd
const userNames = readUserNames()
```

В тестах читать **_/etc/passwd_** нельзя, потому что содержимое этого файла зависит от окружения, в котором запущены тесты. Для тестирования нужно создать файл аналогичной структуры в фикстурах и указать его при запуске функции:

```js
import fs from 'fs'
import path from 'path'

const getFixturePath = filename => path.resolve(**dirname, '../__fixtures__/', filename)

test('readUserNames', () => {
// ../__fixtures__/passwd
const passwdPath = getFixturePath('passwd')
const userNames = readUserNames(passwdPath)
expect(userNames).toEqual(/* ожидаемый список */)
})
```

### Запись файлов

С записью файлов уже сложнее. Главная проблема — отсутствие гарантированной идемпотентности. Это значит, что повторный вызов функции, записывающей файлы, может вести себя не как первый вызов, например, завершаться с ошибкой, либо приводить к другим результатам.

Почему? Представьте себе, что мы пишем тесты на функцию `log(message)`, которая дописывает все переданные в нее сообщения в файл:

```js
const log = makeLogger('development.log')
await log('first message')
// cat development.log
// first message
await log('second message')
// cat development.log
// first message
// second message
```

Это значит, что каждый запуск тестов будет немного другим. При первом запуске тестов создается файл для хранения логов. Затем он начнет заполняться. Это приводит к целой пачке проблем:

- Наверняка внутри этой функции процесс создания файла это особый случай, который нужно тестировать отдельно. Повторные запуски тестов перестанут проверять эту ситуацию.
- Сложнее написать предсказуемый тест. Придется дополнительно придумывать хитрые схемы, например проверять только последнюю строку в файле. Такой подход понижает качество теста.
- Не особенно критично, но все же: в процессе запуска тестов появляется файл, который постоянно растет в размерах.

При правильной организации тестов, каждый тест работает в идентичном окружении на каждом запуске. Для этого, например, можно удалять файл после выполнения каждого теста. В Jest есть хук `afterEach` который выполняется после каждого теста. Эту задачу можно попробовать решить с его помощью:

```js
import fs from 'fs'

test('log', async () => {
  const log = makeLogger('development.log')

  await log('first message')
  const data1 = await fs.readFile('development.log', 'utf-8')
  expect(data1).toEqual(/* ... */)

  await log('second message')
  const data2 = await fs.readFile('development.log', 'utf-8')
  expect(data2).toEqual(/* ... */)
})

afterEach(async () => {
  await fs.unlink('development.log')
})
```

В большинстве ситуаций такое решение работает нормально, но все же не во всех. Выполнение кода тестов — это не атомарная операция. Нет никакой гарантии, что хук `afterEach()` выполнится. Есть много причин, по которым это может не произойти, например, из-за самого Jest.

Есть только один надежный способ делать очистку — делать это до теста, а не после, в `beforeEach()`. С таким подходом есть только одна небольшая сложность. При первом запуске тестов файла нет. Это значит, что прямой вызов `unlink()` завершится с ошибкой и тесты не смогут выполниться. Чтобы избежать этого, можно подавить ошибку:

```js
import _ from 'lodash'

beforeEach(async () => {
  await fs.unlink('development.log').catch(_.noop)
})
```

Другой вопрос при записи файлов — куда их сохранять? Однозначно избегайте записи файлов прямо внутри проекта. Если тестируемый код позволяет сконфигурировать место записи, то используйте системную временную директорию. Ее можно получить через модуль **_os_**:

```js
import os from 'os'

console.log(os.tmpdir())
```

### Виртуальная файловая система (ФС)

Это еще один способ тестировать код, работающий с ФС. С помощью [специальной библиотеки](https://github.com/tschaub/mock-fs) во время тестов создается виртуальная файловая система. Она автоматически подменяет реальную файловую систему для модуля _fs_. Это значит, что функцию, которая тестируется, трогать не надо. Эта функция продолжает думать, что она работает с реальным диском. Вся конфигурация при этом задается снаружи:

```js
import mock from 'mock-fs'

// Конфигурация fs
// Любые операции с этими файлами будут происходить в памяти
// без взаимодействия с реальной файловой системой
mock({
  'path/to/fake/dir': {
    'some-file.txt': 'file content here',
    'empty-dir': {
      /** empty directory */
    },
  },
  'path/to/some.png': Buffer.from([8, 6, 7, 5, 3, 0, 9]),
  'some/other/path': {
    /** another empty directory */
  },
})

await fs.unlink('path/to/fake/dir/some-file.txt')
```

Этот способ дает идемпотентность из коробки. Вызов функции `mock` формирует окружение на каждый запуск с нуля. То есть достаточно добавить ее в `beforeEach` и можно приступать к тестированию.

### Tests

#### My

```js
import { fileURLToPath } from 'url'
import os from 'os'
import path from 'path'
import fs from 'fs/promises'
import getFunction from '../functions.js'

const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)
const prettifyHTMLFile = getFunction()

// BEGIN (write your solution here)
const tmpdir = os.tmpdir()

const getFixturesPath = (filename) =>
  path.resolve(__dirname, '../__fixtures__', filename)
const getTempPath = (filename) => path.resolve(tmpdir, filename)

const afterFile = getFixturesPath('after.html')
const beforeFile = getFixturesPath('before.html')
const tempFilePath = getTempPath('temp.html')

test('prettify main feature', async () => {
  const data1 = await fs.readFile(beforeFile, 'utf-8')
  const data2 = await fs.readFile(afterFile, 'utf-8')

  await fs.writeFile(tempFilePath, data1)
  await prettifyHTMLFile(tempFilePath)

  const result = await fs.readFile(tempFilePath, 'utf-8')

  expect(result).toEqual(data2)
})

afterEach(async () => {
  await fs.unlink(tempFilePath)
})
```

#### Teacher

```js
import { fileURLToPath } from 'url'
import os from 'os'
import path from 'path'
import fs from 'fs/promises'
import getFunction from '../functions.js'

const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)
const prettifyHTMLFile = getFunction()

// BEGIN
const getFixturePath = (name) =>
  path.join(__dirname, '..', '__fixtures__', name)

const filename = 'before.html'
const dest = path.join(os.tmpdir(), filename)
const src = getFixturePath(filename)

let expected

beforeAll(async () => {
  expected = await fs.readFile(getFixturePath('after.html'), 'utf-8')
})

beforeEach(async () => {
  await fs.copyFile(src, dest)
})

test('prettifyHTMLFile', async () => {
  await prettifyHTMLFile(dest)
  const actual = await fs.readFile(dest, 'utf-8')
  expect(actual).toBe(expected)
})
```

## Инверсия зависимостей

Далеко не всегда результат работы функции связан с побочным эффектом, как это было в предыдущем уроке. Иногда побочный эффект это просто дополнительное действие, которое скорее мешает протестировать основную логику.

Представьте себе функцию, которая регистрирует пользователя. Она создает запись в базе данных и отправляет приветственное письмо:

```js
const params = {
  email: 'lala@example.com',
  password: 'qwerty',
}
registerUser(params)
```

Эта функция делает много всего, но главное, что нас волнует — правильная регистрация пользователя. Типичная регистрация сводится к добавлению в базу данных записи о новом пользователе. Именно это и нужно проверять — наличие новой записи в базе данных с правильно заполненными данными. А вот возврат функции нам никак не поможет.

Как правило, базу данных в тестах не прячут. В веб-фреймворках она доступна в тестовой среде и работает как обычно. Идемпотентность в ней достигается за счет транзакций. Перед тестом транзакция начинается и после теста откатывается. Благодаря этому каждый тест запускается в идентичном окружении и не важно как он его меняет:

```js
// Гипотетический пример
const ctx = /* connect to db */;
beforeEach(() => ctx.beginTransaction());

test('registerUser', () => {
  // Внутри идет добавление данных в базу
  const id = registerUser({ name: 'Mike' });
  const user = User.find(id);
  expect(user).toHaveProperty('name', 'Mike');
})

// За счет отката база возвращается в исходное состояние
afterEach(() => ctx.rollbackTransaction());
```

А вот с отправкой писем все сложнее. Ее точно делать нельзя, но как добиться такого поведения? Посмотрите на то, как примерно может выглядеть функция регистрации пользователя:

```js
import sendEmail from './emailSender.js'
const registerUser = (params) => {
  const user = new User(params)
  if (user.save()) {
    sendEmail('registration', { user })
    return true
  }
  return false
}
```

Существует несколько подходов, позволяющих отключить отправку в тестах. Самый простой — переменная окружения, которая показывает среду выполнения:

```js
// Выполняем этот код только если мы не в тестовой среде
if (process.env.NODE_ENV !== 'test') {
  sendEmail('registration', { user })
}
```

Несмотря на простоту использования, такой подход считается плохой практикой. Формально, из-за него происходит нарушение абстракции, код начинает знать о том, где он выполняется. Со временем таких проверок становится все больше и код становится грязнее. Более того, если нам все же надо убедиться, что письмо отправляется (с правильными данными!), то мы не сможем этого сделать.

Следующий способ — поддержка режима тестирования внутри самой библиотеки. Например, где-нибудь на этапе инициализации тестов можно сделать так:

```js
// setup.js в jest
import sendEmail from './emailSender.js'

// У этого подхода много разновидностей, начиная от установки флага,
// заканчивая заменой функций в прототипе.
sendEmail.test = true
```

Теперь в любом другом месте, где импортируется и используется функция `sendEmail()`, реальная отправка происходить не будет:

```js
// Ничего не происходит
sendEmail('registration', { user })
// В отличие от первого варианта, прикладной код ни о чем не догадывается
```

Это довольно популярное решение. Обычно информация о том, как правильно включить режим тестирования, находится в официальной документации конкретной библиотеки.

Что делать, если используемая библиотека не поддерживает режим тестирования? Существует еще один, наиболее универсальный способ. Он основан на применении инверсии зависимостей. Программу можно изменить так, чтобы она вызывала функцию `sendEmail()` не напрямую, а принимала ее как параметр:

```js
import sendEmail from './emailSender.js'

// Ставим значение по умолчанию, чтобы не пришлось постоянно указывать функцию
const registerUser = (params, send = sendEmail) => {
  const user = new User(params)
  if (user.save()) {
    send('registration', { user })
    return true
  }
  return false
}
```

Сначала создадим функцию-замену, которая будет имитировать работу реальной функции отправки писем, но без фактической отправки. Внутри этой функции мы можем выполнять какие-то действия, в зависимости от того, что хотим получить. Например, текст письма можно вывести в терминал для удобства отладки. А можем вообще ничего не делать, оставить тело пустым

```js
const fakeSendEmail = (...args) => {
  // Тут выполняем какие-то действия
  // Или вообще ничего не делаем
}
```

Теперь сам тест. Передаем нашу фейковую функцию отправки письма в качестве параметра при регистрации пользователя

```js
test('registerUser', () => {
  const id = registerUser({ name: 'Mike' }, fakeSendEmail)
  const user = User.find(id)
  expect(user).toHaveProperty('name', 'Mike')
})
```

Ее вызов внутри функции `registerUser()` отработает, но письмо отправляться не будет

Такой способ сложнее в реализации, особенно если функция находится глубоко в стеке вызовов. Это значит, что придется прокидывать нужные зависимости через всю цепочку функций сверху вниз. Самих зависимостей может быть много, и чем больше используется инверсия, тем сложнее код. За гибкость приходится платить.

Теперь плюсы. Ни библиотека, ни код ничего не знают про тесты. Этот способ наиболее гибкий, он позволяет задавать конкретное поведение для конкретной ситуации. В некоторых экосистемах инверсия зависимостей определяет процесс сборки приложения. Особенно в мире PHP, Java и C#.

### Tests

#### My

```js
import { fileURLToPath } from 'url'
import path from 'path'
import _ from 'lodash'
import getFunction from '../functions.js'

const getFilesCount = getFunction()

const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)
const getFixturePath = (name) =>
  path.join(__dirname, '..', '__fixtures__', name)

// BEGIN (write your solution here)
const emptyF = () => {
  return
}

test('getFilesCount main flow', async () => {
  const flatres = await getFilesCount(getFixturePath('flat'), emptyF)
  const nestedres = await getFilesCount(getFixturePath('nested'), emptyF)

  expect(flatres).toBe(3)
  expect(nestedres).toBe(4)
})
```

#### Teacher

```js
import { fileURLToPath } from 'url'
import path from 'path'
import _ from 'lodash'
import getFunction from '../functions.js'

const getFilesCount = getFunction()

const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)
const getFixturePath = (name) =>
  path.join(__dirname, '..', '__fixtures__', name)

// BEGIN
test('getFilesCount', () => {
  // flat можно не тестировать так как nested покрывает и flat тоже
  const directoryPath = getFixturePath('nested')
  const filesCount = getFilesCount(directoryPath, _.noop)
  expect(filesCount).toBe(4)
})
```

---

```js
_.noop()
```

Lodash

This method returns undefined.

---

## Тестирование HTTP-запросов

Инверсия зависимостей крайне мощная техника, которая работает не только с функциями, но и с объектами. Рассмотрим ее глубже на примере HTTP-запросов и познакомимся с таким понятием как заглушка (стабинг, stub).

Предположим, что у нас есть функция, которая анализирует приватные репозитории организации на GitHub и возвращает те, что являются форками (репозитории, отпочкованные от основного репозитория):

```js
// Библиотека для работы с GitHub API
import { Octokit } from '@octokit/rest'

const getPrivateForksNames = async (username) => {
  const client = new Octokit()
  // Клиент выполняет запрос на гитхаб и возвращает список приватных репозиториев указанной организации
  // Данные хранятся в свойстве data возвращаемого ответа
  const { data } = await client.repos.listForOrg({
    username,
    type: 'private',
  })
  // Оставляем только имена форков
  return data.filter((repo) => repo.fork).map((repo) => repo.name)
}
```

Давайте ее протестируем. Что мы хотим от этой функции? В первую очередь убедиться, что она работает правильно — возвращает массив приватных форков. Идеальный тест выглядел бы так:

```js
test('getPrivateForksNames', async () => {
  const privateForks = await getPrivateForksNames('hexlet')
  expect(privateForks).toEqual([
    /* массив имен, которые мы ожидаем увидеть */
  ])
})
```

К сожалению, не все так просто. Внутри функции выполняется HTTP-запрос. Прикинем, какие проблемы из-за этого могут возникнуть:

1. Нестабильная сеть может тормозить выполнение тестов и приводить к фантомным ошибкам. Тесты будут иногда проходить, иногда нет
2. У сервисов подобных github.com установлены ограничения на запросы в секунду, в час, день и так далее. Со 100% вероятностью тесты начнут упираться в эти лимиты. Более того, есть шанс, что машина с которой идут запросы, будет заблокирована
3. Реальные данные на GitHub не статичны, они могут и, скорее всего, будут меняться, что опять же приведет к ошибкам и необходимости править тесты

### Инверсия зависимостей

Для использования инверсии зависимости добавим вторым аргументом функции сам клиент _Octokit_. Это позволит подменить его в тестах:

```js
import { Octokit } from '@octokit/rest'

// Библиотека передается снаружи и ее можно подменить
const getPrivateForksNames = async (username, client = new Octokit()) => {
  // ...
}
```

Нам придется реализовать фейковый (ненастоящий) клиент, который ведет себя примерно так же, как и реальный _Octokit_, за исключением того, что он не выполняет сетевых запросов. Также нам нужно описать конкретные данные, которые вернет вызов _listForOrg_. Только в таком случае мы сможем протестировать, что функция `getPrivateForksNames()` работает корректно.

```js
// Структура этого класса описывает только ту часть,
// которая необходима для вызова await client.repos.listForOrg(...)
class OctokitFake {
  // Здесь мы описываем желаемые данные, которые вернутся в тесте
  constructor(data) {
    this.data = data
  }

  repos = {
    listForOrg: () => {
      // Структура возврата должна совпадать с реальным клиентом
      // Только в этом случае получится прозрачно подменить реальный клиент на фейковый
      return Promise.resolve({ data: this.data }) // так как метод асинхронный
    },
  }
}
```

И сам тест с использованием этого клиента:

```js
import OctokitFake from '/OctokitFake.js';

test('getPrivateForksNames', async () => {
  const data = /* ответ от GitHub, который мы хотим проверить */;
  const client = new OctokitFake(data);
  // Внутри выполняется запрос, который возвращает сформированные выше данные
  const username = /* имя пользователя на гитхабе */;
  const privateForks = await getPrivateForksNames(username, client);
  expect(privateForks).toEqual(/* что мы ожидаем основываясь на том, что вернул listForOrg */);
});
```

В тестировании для подобных фейковых объектов (или функций) есть специальное название — стаб (stub). Стаб заменяет реальный объект или функцию, позволяя избежать выполнения побочных эффектов или сделать код детерминированным. Стаб не используется для проверки чего-либо, он лишь позволяет изолировать ту часть, которая мешает тестированию основной логики.

### Запрет HTTP-запросов

Другим способом предотвращения HTTP-запросов из тестов является их выключение в тестах. В следующих уроках мы познакомимся с библиотекой Nock, имеющей метод для запрета любых HTTP-запросов из кода: `nock.disableNetConnect()`. Рекомендуется вызывать его в начале файла с тестами. Помимо этого, он помогает увидеть, на какие страницы выполняют запросы сторонние библиотеки. Вот как выглядит вывод после выключения внешних соединений (при условии, что не выполнялась подмена запросов):

```
    HttpError: request to https://api.github.com/orgs/hexlet/repos?type=private failed, reason: Nock: Disallowed net connect for "api.github.com:443/orgs/hexlet/repos?type=private"
```

### Tests

#### My

```js
export default class OctokitFake {
  constructor(data) {
    this.data = data
  }

  repos = {
    listForUser: ({ username }) => {
      return Promise.resolve({ data: this.data })
    },
  }
}
```

```js
import OctokitFake from '../support/OctokitFake.js'
import getFunction from '../functions.js'

const getUserMainLanguage = getFunction()

// BEGIN (write your solution here)
test('main feature', async () => {
  const data = [
    { language: 'php' },
    { language: 'javascript' },
    { language: 'typescript' },
    { language: 'javascript' },
  ]

  const client = new OctokitFake(data)

  const result = await getUserMainLanguage('kyryl', client)

  expect(result).toBe('javascript')
})

test('null', async () => {
  const data = []
  const client = new OctokitFake(data)

  const result = await getUserMainLanguage('kyr', client)

  expect(result).toBe(null)
})
```

#### Teacher

```js
export default class OctokitFake {
  constructor(data) {
    this.data = data
  }

  repos = {
    listForUser: () => Promise.resolve({ data: this.data }), // так как метод асинхронный
  }
}
```

```js
import OctokitFake from '../support/OctokitFake.js'
import getFunction from '../functions.js'

const getUserMainLanguage = getFunction()

// BEGIN
test('getUserMainLanguage', async () => {
  const data = [
    { language: 'ruby' },
    { language: 'php' },
    { language: 'java' },
    { language: 'php' },
    { language: 'js' },
  ]
  const client = new OctokitFake(data)
  const mainLanguage = await getUserMainLanguage('linus', client)
  expect(mainLanguage).toEqual('php')
})

test('getUserMainLanguage when empty', async () => {
  const client = new OctokitFake([])
  const mainLanguage = await getUserMainLanguage('user-without-repos', client)
  expect(mainLanguage).toBeNull()
})
```

## Манкипатчинг

В некоторых ситуациях инверсия зависимостей подходит идеально, в других из-за нее код становится значительно сложнее и иногда запутаннее, особенно если зависимости требуются где-то глубоко в стеке вызовов (так как придется пробрасывать зависимость через все промежуточные функции). Но есть способ, который позволяет добраться до нужных вызовов и изменить их даже без инверсии зависимостей.

Ниже пример класса, в котором метод делает запрос по сети:

```js
import axios from 'axios'

class Api {
  // ...
  async makeRequest(url) {
    const { data } = await axios.get(url)
    this.data = data
  }
}
```

Прототипная модель JS позволяет менять поведение объектов без прямого доступа к ним. Для этого достаточно заменить методы в прототипе. После этого любой объект, имеющий этот прототип, в любой части программы начнет использовать новую реализацию метода.

Используем этот подход, чтобы не дать приложению делать реальный сетевой запрос. Для этого подменим метод:

```js
// Подменяем метод makeRequest так, чтобы он не делал сетевой запрос
// После выполнения этого кода Api меняет свое поведение не только
// в этом модуле, но и вообще по всей программе
Api.prototype.makeRequest = function () {
  console.log('no request')
  this.data = 'test data'
}

// Где-то в другом файле
// Так как объекты передаются по ссылке, то это тот же Api
// что и в коде выше
import Api from './api.js'

const client = new Api()

// Вызывается подмененный makeRequest
client.makeRequest(/* аргументы не важны, внутри они не используются */)
// => 'no request'
```

В тех случаях, когда объект (например, функция-конструктор) используется напрямую, все еще проще, чем с конструктором. Достаточно поменять свойство самого объекта:

```js
Array.isArray('') // false

// Этот код может быть вызван в любом месте программы
Array.isArray = () => true

Array.isArray('') // true

// То же самое касается любого импортируемого объекта
import Api from './api.js'

// Теперь везде, где будет импортироваться Api, это будет измененный Api
Api.boom = () => console.log('Hexlet Magic')

// В любом другом модуле
Api.boom() // => 'Hexlet Magic'
```

Такой подход, когда глобально подменяются значения свойств, называется манкипатчинг ([monkey patching](https://ru.wikipedia.org/wiki/Monkey_patch)). Он считается плохой практикой при написании обычного кода в JS, но он очень популярен и удобен в тестах.

Самый известный пример в JavaScript-мире — библиотека nock. С ее помощью перекрывают реальные сетевые запросы, выполняемые модулем http, который включен в стандартную библиотеку Node.js.

```js
// Пример http-запроса с использованием модуля http
import http from 'http'

const options = {
  hostname: 'hexlet.io',
  port: 443,
  path: '/my',
  method: 'GET',
}

// request — асинхронный метод
const req = http.request(options, (res) => {
  // Тут обрабатываем http-ответ
})
```

И пример использования:

```js
import nock from 'nock'
import { getPrivateForkNames } from '../src.js'

test('getPrivateForkNames', async () => {
  nock(/api\.github\.com/) // это регулярное выражение чтобы не указывать полный адрес
    // get — для GET-запросов, post — для POST-запросов и так далее
    .get(/\/orgs\/hexlet\/repos/)
    .reply(200, [
      { fork: true, name: 'one' },
      { fork: false, name: 'two' },
    ])

  const names = await getPrivateForkNames('hexlet')
  expect(names).toEqual(['one'])
})
```

Цепочка `nock(domain).get(url)` задает полный адрес страницы, запрос к которой надо перехватить. Nock анализирует все выполняемые запросы и подменяет только тот, который соответствует данным параметрам. Домен и адрес страницы могут указываться как целиком, так и через регулярное выражение, чтобы не писать слишком много.

Метод `reply(code, body, headers)` описывает ответ, который нужно вернуть по данному запросу. В самом простом случае достаточно указать код возврата. В нашей же ситуации кроме кода нужны данные. Именно на этих данных мы и проверяем работу функции `getPrivateForkNames()`.

Здесь мы рассмотрели только самый базовый вариант использования Nock. У этой библиотеки огромная документация и множество вариантов использования. Полезно периодически ее просматривать в поисках более элегантных путей решения задач тестирования.

В чем плюсы и минусы такого способа работы?

Главный плюс в том, что такой способ тестирования практически универсальный. Его можно использовать с любым кодом, без необходимости править сам код. Программа даже не будет догадываться, что ее тестируют.

Минус заключается в том, что тестирование черным ящиком превращается в тестирование прозрачным ящиком. Это значит, что тест знает про устройство тестируемого кода и зависит от внутренностей. Такое знание делает тесты хрупкими. Функция может измениться без потери работоспособности, но тесты придется переписывать, потому что они завязаны на конкретные значения домена, страниц и формата возвращаемых данных.

В большинстве ситуаций это не так критично. Поэтому смело используйте Nock в своих проектах, но не забывайте и про другие способы.

### Tests

#### My

```js
import nock from 'nock'
import getFunction from '../functions.js'

const getUserMainLanguage = getFunction()

// nock.disableNetConnect()

// BEGIN (write your solution here)
test('main feature', async () => {
  nock('https://api.github.com')
    .get('/users/kyryl/repos')
    .reply(200, [
      { language: 'java' },
      { language: 'php' },
      { language: 'java' },
    ])

  const data = await getUserMainLanguage('kyryl')
  expect(data).toEqual('java')
})

test('null', async () => {
  nock('https://api.github.com').get('/users/emptyuser/repos').reply(200, [])

  const result = await getUserMainLanguage('emptyuser')
  expect(result).toBeNull()
})
```

#### Teacher

```js
import nock from 'nock'
import getFunction from '../functions.js'

const getUserMainLanguage = getFunction()

nock.disableNetConnect()

// BEGIN
test('getUserMainLanguage', async () => {
  const data = [
    { language: 'ruby' },
    { language: 'php' },
    { language: 'java' },
    { language: 'php' },
    { language: 'js' },
  ]
  nock(/api\.github\.com/)
    .get('/users/linus/repos')
    .reply(200, data)

  const mainLanguage = await getUserMainLanguage('linus')
  expect(mainLanguage).toEqual('php')
})

test('getUserMainLanguage when empty', async () => {
  nock(/api\.github\.com/)
    .get('/users/user-without-repos/repos')
    .reply(200, [])

  const mainLanguage = await getUserMainLanguage('user-without-repos')
  expect(mainLanguage).toBeNull()
})
```

## Моки

До этого момента мы рассматривали побочные эффекты как помеху тестирования нашей логики. Для их изоляции использовались стабы или прямое выключение логики в тестовой среде. После этого можно было спокойно проверять правильность работы функции.

В некоторых ситуациях требуется кое-что другое. Не результат работы функции, а то, что она выполняет нужное нам действие, например, шлет правильный HTTP-запрос с правильными параметрами. Для этого понадобятся моки. Моки проверяют, как выполняется код.

### HTTP

import nock from 'nock'
import { getPrivateForkNames } from '../src.js'

```js
// Предотвращение случайных запросов
nock.disableNetConnect()

test('getPrivateForkNames', async () => {
  // Полное название домена
  const scope = nock('https://api.github.com')
    // Полный путь
    .get('/orgs/hexlet/repos/?private=true')
    .reply(200, [
      { fork: true, name: 'one' },
      { fork: false, name: 'two' },
    ])

  await getPrivateForkNames('hexlet')
  // Метод `scope.isDone()` возвращает `true` только тогда,
  // когда соответствующий запрос был выполнен внутри `getPrivateForkNames`
  expect(scope.isDone()).toBe(true)
})
```

Это и называется мокинг. Мок проверяет, что какой-то код выполнился определенным образом. Это может быть вызов функции, HTTP-запрос и тому подобное. Задача мока убедиться в том, что это произошло, и в том, как конкретно это произошло, например, что в функцию были переданы конкретные данные.

Что дает нам такая проверка? В данном случае — мало что. Да, мы убеждаемся, что вызов был, но само по себе это еще ни о чем не говорит. Так когда же полезны моки?

Представьте, что мы бы разрабатывали библиотеку @octokit/rest, ту самую, что выполняет запросы к GitHub API. Вся суть этой библиотеки в том, чтобы выполнить правильные запросы с правильными параметрами. Поэтому там нужно обязательно проверять выполнение запросов с указанием точных URL-адресов. Только в таком случае можно быть уверенными, что она выполняет верные запросы.

В этом ключевое отличие мока от стаба. Стаб устраняет побочный эффект, чтобы не мешать проверке результата работы кода, например, возврату данных из функции. Мок фокусируется на том, как конкретно работает код, что он делает внутри. При этом чисто технически мок и стаб создаются одинаково, за исключением того, что на мок вешают ожидания, проверяющие вызовы. Это приводит к путанице, потому что часто моками называют стабы. С этим ничего уже не поделать, но для себя всегда пытайтесь понять, о чем идет речь. Это важно, от этого зависит фокус тестов.

Функции
Моки довольно часто используют с функциями (методами). К примеру, они могут проверять:

- Что функция была вызвана
- Сколько раз она была вызвана
- Какие аргументы мы использовали
- Сколько аргументов было передано в функцию
- Что вернула функция

Предположим, что мы хотим протестировать функцию `forEach`. Она вызывает колбек для каждого элемента коллекции:

```js
;[1, 2, 3].forEach((v) => console.log(v)) // или проще [1, 2, 3].forEach(console.log)
```

Так как мы изучаем Jest, то для создания моков воспользуемся встроенным механизмом Jest. В других фреймворках могут быть свои встроенные механизмы. Кроме того, как мы убедились выше, существуют специализированные библиотеки для моков и стабов.

```js
test('forEach', () => {
  // Моки функций в Jest создаются с помощью функции jest.fn
  // Она возвращает функцию, которая запоминает все свои вызовы и переданные аргументы
  // Потом это используется для проверок
  const callback = jest
    .fn()

    [(1, 2, 3)].forEach(callback)

  // Теперь проверяем, что она была вызвана с правильными аргументами нужное количество раз
  expect(callback.mock.calls).toHaveLength(3)
  // Первый аргумент первого вызова
  expect(callback.mock.calls[0][0]).toBe(1)
  // Первый аргумент второго вызова
  expect(callback.mock.calls[1][0]).toBe(2)
  // Первый аргумент третьего вызова
  expect(callback.mock.calls[2][0]).toBe(3)
})
```

С помощью моков мы проверили, что функция была вызвана ровно три раза, и ей, последовательно для каждого вызова, передавался новый элемент коллекции. В принципе, можно сказать, что этот тест действительно проверяет работоспособность функции `forEach()`. Но можно ли сделать это проще, без мока и без завязки на внутреннее поведение? Оказывается, можно. Для этого достаточно использовать замыкание:

```js
test('forEach', () => {
  const result = []
  const numbers = [1, 2, 3]
  numbers.forEach((x) => result.push(x))
  expect(result).toEqual(numbers)
})
```

### Обьекты

Jest позволяет создавать моки и для объектов. Они создаются с помощью функции `jest.spyOn()`, напоминающей уже известную нам `jest.fn()`. Эта функция принимает на вход объект и имя метода в этом объекте, и отслеживает вызовы этого метода. Отследим, в качестве примера, вызов `console.log()`:

```js
test('logging something', () => {
  const spy = jest.spyOn(console, 'log')

  console.log(12)

  expect(spy).toHaveBeenCalled() // => true, т.к. метод log был вызван
  expect(spy).toHaveBeenCalledTimes(1) // => true, т.к. метод был вызван 1 раз
  expect(spy).toHaveBeenCalledWith(12) // true, т.к. log был вызван с аргументом 12
  expect(spy.mock.calls[0][0]).toBe(12) // проверка, идентичная предыдущей
})
```

Кроме того, Jest позволяет создавать свою реализацию сбора данных о вызовах отслеживаемой функции при помощи метода `mockImplementation(fn)`. Колбэком этого метода будет функция, которая выполняется после каждого вызова отслеживаемой функции. Возьмем предыдущий пример, но теперь соберем аргументы каждого вызова `console.log()` в отдельный массив:

```js
test('logging something', () => {
  const logCalls = []
  // Здесь функция внутри mockImplementation принимает на вход аргумент(ы),
  // с которым был вызван console.log, и сохраняет его в заранее созданный массив
  const spy = jest
    .spyOn(console, 'log')
    .mockImplementation((...args) => logCalls.push(args))

  console.log('one')
  console.log('two')

  expect(logCalls.join(' ')).toBe('one two')
})
```

### Преимущества и недостатки

Несмотря на то, что существуют ситуации, в которых моки нужны, в большинстве ситуаций их нужно избегать. Моки слишком много знают о том, как работает код. Любой тест с моками из черного ящика превращается в белый ящик. Повсеместное использование моков приводит к двум вещам:

- После рефакторинга приходится переписывать тесты (много тестов!), даже если код работает правильно. Происходит это из-за завязки на то, как конкретно работает код
- Код может перестать работать, но тесты проходят, потому что они сфокусированы не на результатах его работы, а на том, как он устроен внутри

Там, где возможно использование реального кода, используйте реальный. Там, где возможно убедиться в работе кода без моков, делайте это без моков. Излишний мокинг делает тесты бесполезными, а стоимость их поддержки высокой. Идеальные тесты — тесты методом черного ящика.

---

`jest.fn()` создан чтобы полностью подменять на свою реализацию (мокинг), то есть если вы, например, не хотите, чтобы вызывались какие-то реальные запросы в базу данных при тестировании или какие-то другие побочные эффекты.

`jest.spyOn()` же позволяет просто отслеживать что функция была вызвана, без подмены реализации этой функции. `jest.spyOn()` так же умеет и подменять реализацию (как и `jest.fn()`), но он умеет это делать на время. То есть, если вы хотите сохранить оригинальную реализацию функции и может быть вызывать её при каких-то сценариях, тот тут понадобится `jest.spyOn()`.

На самом деле `jest.spyOn()` - это просто синтаксический сахар. То же самое мы можем сделать используя базовый метод `jest.fn()`. Для этого нужно сохранять базовую реализацию и подменять её обратно, когда это нужно:

```js
import app from './app'
import math from './math'

test('test add', () => {
  // сохраняем базовую реализацию
  const original = math.add

  const addMock = jest.spyOn(math, 'add')

  // подменяем реализацию
  math.add = jest.fn(originalAdd)
  math.add.mockImplementation(() => 'mock')
  expect(app.doAdd(1, 2)).toEqual('mock')
  expect(math.add).toHaveBeenCalledWith(1, 2)

  // восстанавливаем реализацию на оригинальную
  math.add = original
  expect(app.doAdd(1, 2)).toEqual(3)
})
```

---

### Tests

#### My

```js
import { fileURLToPath } from 'url'
import path from 'path'
import { jest } from '@jest/globals'
import getFunction from '../functions.js'

const getFilesCount = getFunction()

const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)
const getFixturePath = (name) =>
  path.join(__dirname, '..', '__fixtures__', name)

test('go!', () => {
  const gg = jest.fn()
  const path = getFixturePath('nested')

  const test1 = getFilesCount(path, gg)

  expect(gg).toHaveBeenCalledTimes(1)
  expect(gg).toHaveBeenCalledWith('Go!')
})
```

#### Teacher

```js
import { fileURLToPath } from 'url'
import path from 'path'
import { jest } from '@jest/globals'
import getFunction from '../functions.js'

const getFilesCount = getFunction()

const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)
const getFixturePath = (name) =>
  path.join(__dirname, '..', '__fixtures__', name)

// BEGIN
test('getFilesCount', () => {
  const directoryPath = getFixturePath('nested')
  const mock = jest.fn()
  getFilesCount(directoryPath, mock)
  expect(mock).toHaveBeenCalledTimes(1)
  expect(mock).toHaveBeenCalledWith('Go!')
})
```

## Property-based тестирование

Property-based тестирование (тестирование, основанное на свойствах) — подход к функциональному тестированию, который помогает проверить, соответствует ли тестируемая функция заданному свойству. Для этого подхода не нужно задавать все тестовые примеры. Его задача — сопоставлять характеристики на выходе с заданным свойством.

Свойство — это утверждение, которое в виде псевдокода можно представить так:

```js
for all (x, y, ...)
such as precondition(x, y, ...) holds
property(x, y, ...) is true
```

Мы описываем инвариант в стиле «для любых данных, таких, что ... выполняется условие ...» и, в отличие от обычных тестов, не задаем явно все тестовые примеры, а только описываем условия, которым они должны удовлетворять.

Предположим, у нас есть функция `divide()`, которая находит частное двух чисел.

```js
const divide = (a, b) => a / b
```

Напишем обычный тест на эту функцию:

```js
const { equal } = require('assert')

equal(divide(4, 2), 2)
equal(divide(18, 3), 6)
```

Здесь все просто, передаем на вход числа 18 и 3, ожидаем получить 6. Тесты проходят. Но здесь мы проверили работу функции только на двух парах входных данных. Тест показывает, что функция отрабатывает верно только в этих двух случаях. Но может оказаться так, что при другой паре чисел функция работает неожиданно. Чтобы решить эту проблему, нам нужно написать тест, который сфокусируется не на входных и выходных параметрах, а на свойствах в целом. Эти свойства должны быть истинными для любой правильной реализации.

У операции деления есть свойство: дистрибутивность справа. Оно означает, что деление суммы двух чисел `a` и `b` на число c равно сумме `a / c + b / c`.

Используем это в тестах. Не будем завязываться на конкретные значения и для получения тестовых данных используем генератор случайных чисел.

```js
const a = Math.random() * 1000
const b = Math.random() * 1000
const c = Math.random() * 1000

const left = divide(a + b, c)
const right = divide(a, c) + divide(b, c)
```

Будем запускать этот тест много раз в цикле, и в определенный момент получим такую комбинацию тестовых данных, когда все три числа равны нулю. Тест упадет, так как деление нуля на нуль дает NaN, а NaN не равен NaN. Так мы понимаем, что в функцию нужно добавить проверку на нули.

```js
AssertionError [ERR_ASSERTION]: NaN == NaN
```

Мы написали обычный тест, но использовали в нем не взятые из головы, а произвольные значения и получили возможность выполнять тест много раз на разных входных данных. Таким образом мы проверили саму спецификацию (то, что функция должна делать), а не ее поведение в отдельных случаях. Это и есть тестирование на основе свойств — property-based testing.

В реальной жизни никто не гоняет тесты в цикле, подставляя туда значения вручную. Для этого есть готовые фреймворки. Данные генерируются автоматически фреймворком для property-based тестирования на основании описанных свойств. Если после определенного числа прогонов со случайными данными, удовлетворяющими описанию, условие выполняется, тест считается пройденным. Иначе фреймворк завершает тест с ошибкой.

Рассмотрим, в чем заключаются преимущества property-based тестирования:

- Охватывает все возможные данные. Фреймворки автоматически генерируют данные на основании описанных свойств. В теории эта особенность позволяет охватить все возможные типы входных данных: например, весь диапазон строк или целых чисел

- Сокращает тестовый пример в случае сбоя: всякий раз, когда происходит сбой, фреймворк пытается сократить тестовый пример. Например, если условием сбоя является наличие заданного символа в строке, фреймворк должен возвращать строку из одного символа, которая содержит только этот символ. Это серьезное преимущество property-тестирования — в случае сбоя тест прекращает работу на минимальном примере, а не на наборе входных данных

- Воспроизводимость: перед каждым запуском теста создаются начальные значения, благодаря которым в случае сбоя можно воспроизвести проверку на том же наборе данных

Важно отметить, что property-тестирование не заменяет модульного. К нему нужно относиться как к дополнительному уровню тестов, который поможет сократить время на проверку корректности работы кода по сравнению с другими подходами.

### Фреймворки

Идея property-тестирования была впервые реализована во фреймворке QuickCheck в языке Haskell. Для JavaScript тоже есть несколько библиотек, одна из них `fast-check`.

Для ее установки нужно выполнить команду:

```js
npm install fast-check --save-dev
```

Протестируем с ее помощью реализацию функции contains(), которая проверяет, содержится ли подстрока в строке. У строк можно выделить два свойства, которые мы можем использовать:

- Строка всегда содержит саму себя в качестве подстроки

- Строка `a + b + c` всегда содержит свою подстроку `b`, независимо от содержания `a, b и c`

```js
import fc from 'fast-check'

// Тестируемый код
const contains = (text, pattern) => text.indexOf(pattern) >= 0

// Описываем свойства

test('string should always contain itself', () => {
  fc.assert(fc.property(fc.string(), (text) => contains(text, text)))
})

test('string should always contain its substring', () => {
  fc.assert(
    fc.property(fc.string(), fc.string(), fc.string(), (a, b, c) =>
      contains(a + b + c, b)
    )
  )
})
```

Разберем структуру теста подробнее

`fc.assert(<property>(, parameters))` — выполняет тестирование и проверяет, что свойство остается верным для всех созданных библиотекой строк `a, b и c`. Когда происходит сбой, эта строка отвечает за сокращение тестового примера до минимального размера, чтобы упростить задачу пользователю. По умолчанию он выполняет проверку свойств по 100 сгенерированным входным данным.

`fc.property(<...arbitraries>, <predicate>)` — описывает свойство. `arbitraries` — это значения, которые отвечают за построение входных данных, а `predicate` — это функция, которая проверяет входные данные. `predicate` должен либо возвращать логическое значение, либо не возвращать ничего и завершать тест в случае сбоя.

`fc.string()` — генератор строк, который отвечает за создание и сокращение тестовых значений.

При желании можно извлечь сгенерированные значения для проверки свойств, заменив `fc.assert` на `fc.sample`:

```js
fc.sample(
  fc.property(fc.string(), fc.string(), fc.string(), (a, b, c) =>
    contains(a + b + c, b)
  )
)
```

Сгенерированные данные будут выглядеть примерно так:

```js
{a: ") | 2", b: "", c: "$ & RJh %%"}
{a: "\\\" ", b:" Y \\\ "\\\" ", c:" $ S # K3 "}
{a:" $ ", b:" \\\\ cx% wf ", c:" 't4qRA "}
{a:" ", b:" ", c:" n? H. 0% "}
{a:" 6_ # 7 ", b:" b ", c:" 4% E "}
...
```

Теперь попробуем протестировать заведомо неправильную реализацию функции `contains().` Используем ее в качестве примера, чтобы показать, что фреймворк генерирует в случае сбоя и как он сокращает ввод:

```js
const contains = (pattern, text) => text.substr(1).indexOf(pattern) !== -1
```

Фреймворк генерирует определенный набор данных. Как только тест видит сбой, он запускает процесс сокращения. При тестировании примера, приведенного выше, происходит сбой:

```js
Error: Property failed after 20 tests
{ seed: 1783957873, path: "19:1:0:1:1", endOnFailure: true }
Counterexample: [""," ",""]
Shrunk 4 time(s)
Got error: Property failed by returning false
```

Теперь рассмотрим другой вариант реализации, когда predicate не будет возвращать логическое значение, а в случае возникновения ошибки будет завершать тест. Поменяем реализацию тестирования функции `contains()` соответственно:

```js
import fc from 'fast-check'

// Тестируемый код
const contains = (text, pattern) => text.indexOf(pattern) >= 0

// Описываем свойства
test('string should always contain its substring', () => {
  fc.assert(
    fc.property(fc.string(), fc.string(), fc.string(), (a, b, c) => {
      expect(contains(a + b + c, c)).toBeTruthy()
    })
  )
})
```

### Заключение

Тестирование на основе свойств — полезный и мощный инструмент. Мы не должны отказываться от классических тестов, но можем их комбинировать с тестированием на основе свойств. Например, можно базовый функционал покрывать классическими тестами на основе примеров, а критически важные функции дополнительно покрывать property-тестами.

### Tests

#### My

```js
// @ts-check

import fc from 'fast-check'
import getFunction from '../functions.js'

const sort = getFunction()

// BEGIN (write your solution here)
const numbers = [1, 9, 7, 0, 2, 5]

test('fast-check', () => {
  fc.assert(
    fc.property(
      fc.uniqueArray(fc.integer(), { minLength: 5 }, { maxLength: 10 }),
      (numbers) => {
        expect(sort(numbers)).toBeSorted()
      }
    )
  )
})

test('classic', () => {
  expect(sort(numbers)).toBeSorted()
})
```

#### Teacher

```js
// @ts-check

import fc from 'fast-check'
import getFunction from '../functions.js'

const sort = getFunction()

// BEGIN
test('should be sorted', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (data) => {
      const result = sort(data)
      expect(result).toBeSorted()
    })
  )
})
```
