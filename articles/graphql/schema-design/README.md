# Дизайн GraphQL-схем — делаем АПИ удобным, избегаем боль и страдания

Рекомендации и правила озвученные в этой статье были выработаны за 3 года использования GraphQL как на стороне сервера (при построении схем) так и на клиентской стороне (написания GraphQL-запросов и покрытием клиентского кода статическим анализом). Также в этой статье используются рекомендации и опыт Caleb Meredith (автора PostGraphQL, ex-сотрудник Facebook) и иженеров Shopify.

Эта статья может поменяться в будущем, т.к. текущие правила носят рекомендательный характер и могут со временем улучшиться, измениться или вовсе стать антипаттерном. Но то что здесь написано, выстрадано временем и болью от использования кривых GraphQL-схем.

## TL;DR всех правил

- 1. Правила именования
  - 1.1. Используйте `camelCase` для именования GraphQL-полей и аргументов.
  - 1.2. Используйте `UpperCamelCase` для именования GraphQL-типов.
  - 1.3. Используйте `CAPITALIZED_WITH_UNDERSCORES` для именования значений ENUM-типов.
- 2. Правила типов
  - 2.1. Используйте кастомные скалярные типы, если вы хотите объявить поля или аргументы с определенным семантическим значением.
  - 2.2. Используйте Enum для полей, которые содержат определенный набор значений.
- 3. Правила полей (Output)
  - 3.1. Давайте полям понятные смысловые имена, а не то как они реализованы.
  - 3.2. Группируйте взаимосвязанные поля вместе в новый output-тип.
- 4. Правила аргументов (Input)
  - 4.1. Группируйте взаимосвязанные аргументы вместе в новый input-тип.
  - TODO: Rule #20: Use stronger types for inputs (e.g. DateTime instead of String) when the format may be ambiguous and client-side validation is simple. This provides clarity and encourages clients to use stricter input controls (e.g. a date-picker widget instead of a free-text field).
  - TODO: Rule #18: Only make input fields required if they're actually semantically required for the mutation to proceed.
- 5. Правила списков
  - 5.1. Для фильтрации списков используйте аргумент `filter` c типом Input, который содержит в себе все доступные фильтры.
  - 5.2. Для сортировки списков используйте аргумент `sort`, который должен быть массивом перечисляемых значений [Enum!].
  - 5.3. Для ограничения возвращаемых элементов в списке используйте аргументы `limit` со значением по-умолчанию и `skip`.
  - 5.4. Для пагинации используйте аргументы `page`, `perPage` и возвращайте output-тип с полями `items` с массивом элементов и `pageInfo` с мета-данными для удобной отрисовки страниц на клиенте.
  - 5.5. Для бесконечных списков (infinite scroll) используйте [Relay Cursor Connections Specification](https://facebook.github.io/relay/graphql/connections.htm).
- 6. Правила Мутаций
  - 6.1. Используйте Namespace-типы для группировки мутаций в рамках одного ресурса!
  - 6.2. Забудьте про CRUD - cоздавайте небольшие мутации для разных логических операций над ресурсами.
  - 6.3. Рассмотрите возможность выполнения мутаций сразу над несколькими элементами (однотипные batch-изменения).
  - 6.4. У мутаций должны быть четко описаны все обязательные аргументы, не должно быть вариантов либо-либо.
  - 6.5. У мутации вкладывайте все переменные в один уникальный `input` аргумент.
  - 6.6. Мутация должна возвращать свой уникальный Payload-тип.
  - 6.7. В ответе мутации возвращайте поле с типом Query
  - 6.8. Мутации должны возвращать в Payload'e поле `errors` с типизированными пользовательскими ошибками.
  - TODO: Rule #23: Most payload fields for a mutation should be nullable, unless there is really a value to return in every possible error case.
  - TODO: Rule #19: Use weaker types for inputs (e.g. String instead of Email) when the format is unambiguous and client-side validation is complex. This lets the server run all non-trivial validations at once and return the errors in a single place in a single format, simplifying the client.
- 7. Правила реляций между типами (relationships)
  - TODO: Rule #1: Always start with a high-level view of the objects and their relationships before you deal with specific fields.
  - TODO: Rule #8: Always use object references instead of ID fields.
- 8. Правила по бизнес-логике
  - TODO: Rule #2: Never expose implementation details in your API design.
  - TODO: Rule #3: Design your API around the business domain, not the implementation, user-interface, or legacy APIs.
  - TODO: Rule #5: Major business-object types should always implement Node.
  - TODO: Rule #12: The API should provide business logic, not just data. Complex calculations should be done on the server, in one place, not on the client, in many places.
  - TODO: Rule #13: Provide the raw data too, even when there’s business logic around it.
- 9. Правила по версионированию схемы
  - TODO: Rule #4: It’s easier to add fields than to remove them.
  - TODO: Unique payload type. Use a unique payload type for each mutation and add the mutation’s output as a field to that payload type.

---

## 1. Правила именования

GraphQL для проверки имен полей и типов использует следующую регулярку `/[_A-Za-z][_0-9A-Za-z]*/`. Согласно нее можно использовать `camelCase`, `under_score`, `UpperCamelCase`, `CAPITALIZED_WITH_UNDERSCORES`. Слава богу `kebab-case` ни в каком виде не поддерживается.

Так что же лучше выбрать для именования?

Абстрактно можно обратиться к исследованию [Eye Tracking'а](http://www.cs.kent.edu/~jmaletic/papers/ICPC2010-CamelCaseUnderScoreClouds.pdf) по  `camelCase` и `under_score`. В этом исследовании явного фаворита не выявлено.

Коль исследования особо не помогло. Давайте будем разбираться в каждом конкретном случае.

### 1.1. Используйте `camelCase` для именования GraphQL-полей и аргументов.

Названиями полей активнее всего пользуются потребители GraphQL-апи, т.е. наши любымые клиенты — браузеры с JavaScript и разработчики мобильных приложений. Давайте посмотрим, что чаще всего используется по их конвенции для именования переменных. Ведь если клиенты дергают ваше апи, то скорее всего они будут использовать ваше именование для переменных у себя в коде. Ведь маппить (алиасить) название полей в удобный формат не шибко интересная работа.

Согласно [википедии](https://en.wikipedia.org/wiki/Naming_convention_(programming)) следующие клиентские языки (потребители GraphQL апи) придерживаются следующих конвенций по именованию переменных:

- JavaScript — `camelCase`
- Java — `camelCase`
- Swift — `camelCase`
- Kotlin — `camelCase`

Конечно каждый у себя "на кухне" может использовать `under_score`. Но в среднем по больнице  используется `camelCase`. Если найдете какое-нибудь исследование по процентовки использованию `camelCase` и `under_score` в том или ином языке программирования — дайте пожалуйста знать, очень тяжело гуглится вся это тема. Кругом сплошной субъективизм.

А ещё, если залезть в кишки graphql и посмотреть его [IntrospectionQuery](https://github.com/graphql/graphql-js/blob/master/src/utilities/introspectionQuery.js), то он также написан используя `camelCase`.

PS. Мне очень печально видеть в документации MySQL или PostgreSQL примеры с названием полей через `under_score`. Потом все это дело качует в код бэкенда, далее появляется в GraphQL апи, а потом и в коде клиентов. Блин, ведь эти БД спокойно позволяют сразу объявить поля в `camelCase`. Но так как на начале не определились с конвенцией имен, а взяли как в документашке, то потом происходят холивары что `under_score` лучше чем `camelCase`. Ведь переименовывать поля на уже едущем проекте больно и черевато ошибками. Вобщем тот кто выбрал `under_score`, тот и должен маппить поля в `camelCase`!

### 1.2. Используйте `UpperCamelCase` для именования GraphQL-типов.

А вот именование типов, в отличии от полей уже происходит немного по другому.

В самом GraphQL уже есть стандартные скалярные типы `String`, `Int`, `Boolean`, `Float`. Они именуются через `UpperCamelCase`.

Также внутренние типы GraphQL-интроспекции `__Type`, `__Field`, `__InputValue` и пр. Именуются через `UpperCamelCase` с двумя андерскорами в начале.

А еще GraphQL статический типизированный язык запросов. И из GraphQL-запросов много кто генерирует тайп-дефинишены для статического анализа кода. Так вот если посмотреть как в JS именуют сложные типы во Flowtype и TypeScript — то тут тоже обычно используется `UpperCamelCase`.

### 1.3. Используйте `CAPITALIZED_WITH_UNDERSCORES` для именования значений ENUM-типов.

Enum в GraphQL используются для перечисления списка возможных значение у некого типа.

В самом GraphQL для интроспекции в типе `__TypeKind` используются следующие значения: `SCALAR`, `OBJECT`, `INPUT_OBJECT`, `NON_NULL`  и другие. Т.е. используется `CAPITALIZED_WITH_UNDERSCORES`.

Если относиться к Enum-типу как к типу с набором констант. То в большинстве языков программирования константы именуются через `CAPITALIZED_WITH_UNDERSCORES`.

Также `CAPITALIZED_WITH_UNDERSCORES` будет хорошо смотреться в самих GraphQL-запросах, чтобы четко индетифицировать Enum'ы:

```graphql
query {
  findUser(sort: ID_DESC) {
    id
    name
  }
}
```

## 2. Правила типов

GraphQL по спецификации содержит всего 5 типов скалярных полей `String` (строка), `Int` (целое число), `Float` (число с точкой), `Boolean` (булевое значение), `ID` (строка с уникальным индетификатором). Все эти типы легко и понятно передаются через JSON на любом языке программирования.

А вот когда речь заходит о каком-то скалярном типе не входящего в синтаксис JSON, например `Date`. То тут уже необходимо бэкендеру самостоятельно определяться с форматом данных, чтоб их можно было сериализовать и передать через JSON клиенту. А также легко и просто получить значение этого типа от клиента и десериализовать его на сервере.

В таких случаях GraphQL позволяет создавать свои кастомные скалярные типы. О том как это делается [написано здесь](../types/README.md#custom-scalar-types).

### 2.1. Используйте кастомные скалярные типы, если вы хотите объявить поля или аргументы с определенным семантическим значением.

Если ваше поле возвращает тип `String`, то клиентам вашего апи сложно понять какие ограничения или семантическое значение содержиться в этой строке. Например с помощью типа `String` вы можете передавать обычный текст, HTML, строку длиной в 255 символов, строку в base64, дату или любое другое значение в конкретном формате.

Для того чтобы сделать ваше АПИ более прозрачным для команды, создавайте кастомные скалярные типы. Например: `HTML`, `String255`, `Base64`, `DateString`. Бэкендерам и фронтендерам это позволит единожды написать правила валидации и проверки таких типов и не дублировать код в каждом месте где они будут использоваться.

```diff
type Article {
  id: ID!
-  description: String
+  description: HTML
}
```

В таком случае легче понимать, что конкретно прилетает в поле `description`. И понятно что строку не надо эскейпить для отображения в браузере, а показывать как есть.

### 2.2. Используйте Enum для полей, которые содержат определенный набор значений.

Частенько в схемах встречаются поля, которые содержат определенный набор значений. Например: `пол`, `статус заявки`, `код страны`, `вид оплаты`. Использовать просто тип `String` или `Int` в таких случаях никоим образом не помогает вашим клиентам понять, какие значения могут быть получены.

Конечно доступные значения можно описать в документации. Но я не вижу смысла этого делать, когда в GraphQL есть тип `Enum` — [читать подробнее](../types/README.md#enumeration-types).

Значения полей с типом `Enum` проверяются на этапе валидации запроса. А также они могут проверяться на этапе разработки линтерами и статическими анализаторами. Это на порядок полезнее и безопаснее, чем тупо перечислять значения в документашке.

```diff
type User {
  id: ID!
-  gender: String
+  gender: Gender
}

+ enum GenderEnum {
+   MALE
+   FEMALE
+ }
```

## 3. Правила полей (Output)

В GraphQL поля используются для передачи данных с сервера на клиент. Они могут быть описаны обычными скалярнвми типами, а также сложными структурами – Output-типами (GraphQLObjectType).

### 3.1. Давайте полям понятные смысловые имена, а не то как они реализованы.

Необходимо полям давать понятные имена. Это очень простое и банальное правило. К примеру у нас есть следующий тип:

```diff
type Meeting {
  title: String
-  bodyHtml: String
+  description: HTML
}
```

Человек который в первый раз видит тип `Meeting`, будет гадать что конкретно хранится в поле `bodyHtml`. Здорово если бэкендеры не ленятся и оставляют описание к полям. Но черт возьми, можно же поле в АПИ назвать `description`, а в базе пусть хранится как `bodyHtml`, тогда и документашку читать не нужно.

### 3.2. Группируйте взаимосвязанные поля вместе в новый output-тип.

Часто бывают ситуации, когда несколько полей по логике взаимоувязаны друг с другом. К примеру наш `Meeting` может быть 1-1 или групповой, соответственно он имеет разные наборы полей для каждого типа встречи. Так вот, чтобы не делать бардак из полей и явно дать понять фронтендеру какие поля можно использовать для того или иного вида встречи, лучше их разнести по подтипам. Например настройки для 1-1 вчтречи уносим в тип `MeetingSettingsOneToOne`, а настройки групповой встречи в `MeetingSettingsGroup`:

```graphql
type Meeting {
  title: String
  type: MeetingType
  settingsOneToOne: MeetingSettingsOneToOne
  settingsGroup: MeetingSettingsGroup
}

enum MeetingType { ONE_TO_ONE, GROUP }

type MeetingSettingsOneToOne {
  peer1: String!
  peer2: String!
}

type MeetingSettingsGroup {
  host: String!
  maxAtendees: Int!
  accessKey: String!
  canSeePreviousMessages: Boolean!
  greetingMessage: String
}
```

Обратите внимание, что в зависимости от типа встречи присутствует разный набор обязательных полей в под-типах `MeetingSettingsOneToOne` и `MeetingSettingsGroup`. Если бы мы все эти поля объявили на верхнем уровне в типе `Meeting`, то мы бы не смогли их сделать обязательными, что сделало бы нашу схему менее строгой. А также имели бы бардак из разных полей, которые необходимо было бы специально документировать для фронтендера.

Т.е. если сгруппировать взаимоувязанные поля в под-тип, то это делает схему не только легче для восприятия, но и позволяет достаточно легко ее расширять в будущем.

## 4. Правила аргументов

TODO: some rules

- Rule #20: Use stronger types for inputs (e.g. DateTime instead of String) when the format may be ambiguous and client-side validation is simple. This provides clarity and encourages clients to use stricter input controls (e.g. a date-picker widget instead of a free-text field).
- Rule #18: Only make input fields required if they're actually semantically required for the mutation to proceed.

## 5. Правила списков

Я не встречал ни одного АПИ, которое бы не возвращало список элементов. Либо это постраничная листалка, либо что-то построенное на курсорах для бесконечных списков. Списки надо фильтровать, сортировать, ограничивать кол-во возвращаемых элементов. Сам GraphQL никак не ограничивает свободу реализации, но для того чтобы сформировать некое единообразие, необходимо завести стандарт.

### 5.1. Для фильтрации списков используйте аргумент `filter` c типом Input, который содержит в себе все доступные фильтры.

Как вы думаете, как лучше организовать фильтрацию?

```graphql
type Query {
  articles(authorId: Int, tags: [String], lang: LangEnum): [Article]
}
```

или через аргумент `filter` с типом `ArticleFilter`:

```graphql
type Query {
  articles(filter: ArticleFilter): [Article]
}

input ArticleFilter {
  authorId: Int
  tags: [String]
  lang: LangEnum
}
```

Конечно, лучше всего организовать через дополнительный тип `ArticleFilter`. На это есть несколько причин:

- если вы будете добавлять новые аргументы не относящиеся к фильтрации (сортировка, лимит, офсет, номер страницы, курсор, язык и прочее), то ваши агрументы не будут путаться друг с другом
- на клиенте для статического анализа вы получите `ArticleFilter` тип. Иначе клиенты будут вынуждены собирать такой тип вручную, что черевато ошибками
- тупо легче читать и воспринимать вашу схему, когда в ней 3-5 аргументов а не 33 аргумента с возможной фильтрацией. Есть аргумент `filter` и если нужно провались в него и там уже посмотри все 33 поля для фильтрации
- этот фильтр можно переиспользовать несколько раз в вашем апи, если список статей можно запросить из нескольких мест

Также важно договориться как назвать поле для фильтрации. А то если у вас 5, 10 или 100 разработчиков, то на выходе в схеме у вас появиться куча названий для аргумента фильтрации — `filter`, `where`, `condition`, `f` и прочий нестандарт. Если учитывать что есть базы SQL и noSQL, есть всякие кэши и прочие сервисы, то **самым адекватным именем для аргумента фильтрации является — `filter`**. Оно понятно и подходит для всех! А вот этот `where` только для SQL-бэкендеров.

### 5.2. Для сортировки списков используйте аргумент `sort`, который должен быть массивом перечисляемых значений [Enum!].

Когда в списке много записей, то может потребоваться сортировка по полю. А иногда требуется сортировка по нескольким полям.

Для начала команде необходимо выбрать имя для аргумента сортировки. На ум приходит следующие популярные названия — `sort`, `order`, `orderBy`. Т.к. слово `order` переводиться не только как порядок, но и как заказ; и используется в основном только в реляционных базах данных. **То лучшим выбором имени для поля сортировки будет — `sort`.** Оно однозначно трактуется и будет понятно всем.

Когда с именем аргумента определились, то необходимо выбрать тип для аргумента сортировки:

- Если взять `String`, то фронтендеру будет тяжело указать правильные значения и при этом мы не получаем возможности валидации параметров средствами GraphQL.
- Можно создать input-тип `input ArticleSort { field: SortFieldsEnum, order: AscDescEnum }` — структура у которой можно выбрать имя поля и тип сортировки. Но такой подход не подойдет, если у вас появится полнотекстовая сортировка или сортировка по близости. У них просто нет значения DESC (обратной сортировки).
- Остается один самый простой и верный способ — использовать Enum для перечисления списка доступных сортировок `enum ArticleSort { ID_ASC, ID_DESC, TEXT_MATCH, CLOSEST }`. Т.е. в таком случае вы можете явно указать доступные способы для сортировки.

Также если внимательно прочитать как [объявляются типы Enum](../types/README.md#enumeration-types) на сервере, то помимо ключа `ID_ASC`, можно задать значение `id ASC` (для SQL), либо `{ id: 1 }` (для NoSQL). Т.е. клиенты видят унифицированный ключ `ID_ASC`, а вот на сервере в resolve-методе вы получаете уже готовое значение для подстановки в запрос. Конвертация ключа сортировки происходит внутри Enum-типа, что в свою очередь сделает код вашего resolve-метод меньше и чище.

Ну а теперь, чтобы иметь возможность сортировать по нескольким полям, нам просто необходимо дать возможность передавать массив значений для сортировки. В итоге получим следующее объявление сортировки:

```graphql
type Query {
  articles(sort: [ArticleSort!]): [Article]
}

enum ArticleSort {
  ID_ASC, ID_DESC, TEXT_MATCH, CLOSEST
}
```

### 5.3. Для ограничения возвращаемых элементов в списке используйте аргументы `limit` со значением по-умолчанию и `skip`.

С ограничением кол-ва элементов в списке и возможностью сдвига все банально просто. Используйте аргументы с названиями `limit` и `skip`. Единственно для лимита хорошо бы задать значение по-умолчанию, чтоб клиенты могли не указывать это значение.

```graphql
type Query {
  articles(
    limit: Int = 20
    skip: Int
  ): [Article]
}
```

### 5.4. Для пагинации используйте аргументы `page`, `perPage` и возвращайте output-тип с полями `items` с массивом элементов и `pageInfo` с мета-данными для удобной отрисовки страниц на клиенте.

Альтернативой для ограничения возвращаемых элементов в списке `limit` и `skip` может выступить пагинация.

Для пагинации лучше всего использовать аргументы с именами `page` и `perPage` со значениями по-умолчанию:

```graphql
type Query {
  articles(
    page: Int = 1
    perPage: Int = 20
  ): [Article]
}
```

Но если вы остановитесь только на аргументах `page` и `perPage`, то польза от вашей пагинации для клиентов будет ничем не лучше `limit` и `skip`. Для того, чтобы клиентское приложение могло отрисовать нормально пагинацию, ему необходимо предоставить не только сами элементы списка, но и дополнительные метаданные как минимум с общим кол-вом страниц и записей. Для метаданных пагинации можно завести следующий общий тип `PaginationInfo`:

```graphql
type PaginationInfo {
  # Total number of pages
  pageCount: Int

  # Total number of items
  itemCount: Int

  # Current page number
  currentPage: Int!

  # Number of items per page
  perPage: Int!

  # When paginating forwards, are there more items?
  hasNextPage: Boolean

  # When paginating backwards, are there more items?
  hasPreviousPage: Boolean
}
```

В случае предоставления мета-данных для пагинации мы уже не можем взять просто и вернуть массив найденых элементов. Нам необходимо будет завести новый тип `ArticlePagination`. И вот здесь опять появляется повод к выработке стандарта:

```graphql
type Query {
  articles(
    page: Int = 1
    perPage: Int = 20
  ): ArticlePagination
}

type ArticlePagination {
  # Array of objects.
  items: [User]
  
  # Information to aid in pagination.
  pageInfo: PaginationInfo!
}
```

У `ArticlePagination` должно быть как минимум два поля:

- `items` — массив элементов
- `pageInfo` — объект с мета-данными пагинации `pageCount`, `itemCount`, `currentPage`, `perPage`

### 5.5. Для бесконечных списков (infinite scroll) используйте [Relay Cursor Connections Specification](https://facebook.github.io/relay/graphql/connections.htm).

У пагинации есть недостаток, когда добавляются или удаляются элементы, то при переходе на следующую страницу вы можете столкнуться с проблемами:

- under-fetching — это когда в начале списка удаляется элемент, и при переходе на следующую страницу клиент пропускает запись, которая "убежала" на предыдущую страницу
- over-fetching — это когда добавляются новые записи в начало списка, и при переходе на следующую страницу клиент повторно видит записи которые были на предыдущей страницы

Для решения этой проблемы Facebook разработал спецификацию [Relay Cursor Connections Specification](https://facebook.github.io/relay/graphql/connections.htm). Она идеально подходит для создания бесконечных (infinite scroll) списков. А коль есть спецификация, то значит есть некий стандарт которому может следовать команда разработчиков и не изобретать велосипеды.

## 6. Правила Мутаций

Больше всего бардака разводят в Мутациях. Внимательно прочитайте следующие правила, которые позволят вам сделать ваше АПИ сухим, чистым и удобным.

### 6.1. Используйте Namespace-типы для группировки мутаций в рамках одного ресурса.

В большенстве GraphQL-схем страшно заглядывать в мутации. На АПИ среднего размера кол-во мутаций может легко переваливать за 50-100 штук, и это все на одном уровне. Ковыряться и искать нужную операцию в таком списке достаточно сложно.

Shopify рекомендует придерживаться такого именования для мутаций `collection<Action>`. В списке это позволяет хоть как-то сгрупировать операции над одним ресурсом. Кто-то противник такого подхода, и форсит использование `<action>Collection`.

В любом случае есть способ получше – используйте Namespace-типы. Это такие типы которые содержат в себе набор операций над одним ресурсом. Если представить путь запроса в dot-нотации, то выглядит он так `Mutation.<collection>.<action>`.

В NodeJS это делается достаточно легко. Я приведу несколько примеров с использованием разных библиотек:

Стандартная имплементация с пакетом `graphql`:

```js
// Create Namespace type for Article mutations
const ArticleMutations = new GraphQLObjectType({
  name: 'ArticleMutations',
  fields: () => {
    like: { type: GraphQLBoolean, resolve: () => { /* resolver code */ } },
    unlike: { type: GraphQLBoolean, resolve: () => { /* resolver code */ } },
  },
});

// Add `article` to regular mutation type with small magic
const Mutation = new GraphQLObjectType({
  name: 'Mutation',
  fields: () => {
    article: {
      type: ArticleMutations,
      resolve: () => ({}), // ✨✨✨ magic! which allows to proceed call of sub-methods
    }
  },
});
```

С помощью пакета `graphql-tools`:

```js
const typeDefs = gql`
  type ArticleMutations {
    like: Boolean
    unlike: Boolean
  }

  type Mutation {
    article: ArticleMutations
}
`;

const resolvers = {
  Mutation: {
    article: () => ({}), // ✨✨✨ magic! which allows to proceed call of sub-methods
  }
  ArticleMutations: {
    like: () => { /* resolver code */ },
    unlike: () => { /* resolver code */ },
  },
};
```

С помощью `graphql-compose`:

```js
schemaComposer.Mutation.addNestedFields({
  'article.like': { // ✨✨✨ magic! Just use dot-notation with `addNestedFields` method
    type: 'Boolean',
    resolve: () => { /* resolver code */ }
  },
  'article.unlike': {
    type: 'Boolean',
    resolve: () => { /* resolver code */ }
  },
});
```

Соответственно клиенты будут делать такие запросы к вашему АПИ:

```graphql
mutation {
  article {
    like(id: 15)
  }

  ### Forget about ugly mutations names!
  # artileLike
  # likeArticle
}
```

Итак правило, для избежания бардака в мутациях используйте Namespace-типы для группировки мутаций в рамках одного ресурса!

### 6.2. Забудьте про CRUD – cоздавайте небольшие мутации для разных логических операций над ресурсами.

С GraphQL надо отходить от CRUD (create, read, update, delete). Если вешать все измениня на мутацию `update`, то достаточно быстро она станет массивной и тяжелой в обслуживании. Здесь речь идет не о простом редактировании полей заголовок и описание, а о "сложных" операциях.  К примеру, для публикации статей создайте мутации `publish`, `unpublish`. Необходимо добавить лайки к статье - создайте две мутации `like`, `unlike`. Помните, что потребители вашего АПИ слабо представляют себе структуру взаимосвязей ваших данных, и за что каждое поле отвечает. А набор мутаций над ресурсом быстро позволяет фронтендеру "вьехать" в набор возможных операций.

Да и вам в будущем будет легче отслеживать по логам, что пользователи чаще всего дергают. И оптимизировать узкие места.

### 6.3. Рассмотрите возможность выполнения мутаций сразу над несколькими элементами (однотипные batch-изменения).

Клиентские приложения становятся более умными и удобными. Часто пользователю предлагаются batch-операции – добавление нескольких записей, массового удаления или сортировки. Отправлять операции по одной будет накладно. Как-то агрегировать их в сложный GraphQL-запрос с несколькими мутациями, т.е. генерировать один общий запрос на клиенте - порождает кучу кода с душком.

Например: есть мутация `deleteArticle`, добавьте еще `deleteArticles`. Для того чтоб пользователь мог нащелкать несколько статей и подтвердить удаление сразу пачкой.

Но здесь самое главное без фанатизма, не надо на все подряд вешать массовые операции. Всегда руководствуйтесь здравым смыслом.

### 6.4. У мутаций должны быть четко описаны все обязательные аргументы, не должно быть вариантов либо-либо.

К примеру ваше АПИ позволяет отправить разные письма с помощью мутации `sendEmail(type: PASSWORD_RESET, params: JSON)`. Для того чтобы выбрать шаблон, вы передаете Enum аргумент с типом письма и для него передаете какие-то параметры.

Проблема такого подхода в том, что клиент заранее точно не знает какие параметры необходимо передать для того или иного типа писем. К тому же, если в будущем проводить рефакторинг схемы, то статическая типизация нам не позволит отловить ошибки на клиенте.

Лучше разбивать мутации на несколько штук с жестким описанием аргументов. Например: `sendEmailPasswordReset(login: String!, note: String)`. При этом не забываем аргументы помечать как обязательные, если без них операция не отработает.

Также бывают ситуации, когда вы обязаны передать либо один аргумент, либо другой. К примеру, мы можем отправить письмо по сбросу пароля если укажут login или email. В таком случае мы не можем оба аргумента в нашей мутации сделать обязательными. Пользователь узнает только в рантайме, что забыл передать обязательный аргумент. Да и фронтендеру будет не сразу понятно, что надо передавать либо `login`, либо `email`. А что будет если передать оба аргумента от разных пользователей?

В таком случае просто заводится две мутации, где жестко расписаны обязательные аргументы:

- `sendResetPasswordByLogin(login: String!)`
- `sendResetPasswordByEmail(email: String!)`

Не экономьте на мутациях и старайтесь избегать слабой типизации.

### 6.5. У мутации вкладывайте все переменные в один уникальный `input` аргумент.

Старайтесь в мутациях использовать один аргумент `input`. Его гораздо легче использовать на клиентской стороне. Клиенту потребуется передать всего одну переменную, а не вагон для каждого аргумента в мутации.

```graphql
# Good:
mutation ($input: UpdatePostInput!) {
  updatePost(input: $input) { ... }
}

# Not so good – запрос гороздо сложнее писать:
mutation ($id: ID!, $newText: String, ...) {
  updatePost(id: $id, newText: $newText, ...) { ... }
}
```

Если у мутации на верхнем уровне один-два аргумента, то при таком подходе они становятся более стройными и читабельными. При этом без дополнительных затрат, кроме нескольких дополнительных нажатий клавиш, вложение агрументов позволяет вам полностью использовать возможности GraphQL, в качестве version-less API (безверсионного апи). Вложенность дает вам возможность расширять типы с течением времени и избегать конфликтов в именовании полей.

Также при статической типизации с помощью Typescript или Flowtype гораздо легче отследить изменения в вашем АПИ, когда в коде идет привязка к одному сложному-типу, а не набору разрозненных аргументов.

Думайте о вложении аргументов в один общий аргумент `input`, как об инвестиции в будущие изменения вашего GraphQL API:

```graphql
mutation {
  createPerson(input: {
    # Позволяя вкладывать поля в `input` мы даем себе возможноть
    # добавлять поля, такие как `password` или `metadata`.
    # Также мы можем задеприкейтить поле `person`
    # и добавить вместо него `partialPerson`.
    person: {
      id: 4
      name: "Budd Deey"
    },
    password: "qwerty"
  }) { ... }

  updatePerson(input: {
    # Поле `id` чтоб найти нужную запись для изменений
    id: 4,
    # Поле `patch` показывает что мы хотим изменить
    patch: {
      name: "Budd Deey"
    },
    # Поле `append` что мы хотим добавить к существующим значениям
    append: {
      tags: ['winter', 'is', 'coming'],
    }
  }) { ... }
}
```

При этом не экономьте на типах – для каждой мутации заводите свой Input-тип с уникальным именем. Это позволит вам менять мутации, не оглядываясь на то, что новая семантика может поломать другие мутации.

Также по состоянию на конец 2018 года в спецификации GraphQL нет возможности деприкейтить аргументы (помечать их как устаревшие). Но вот деприкейтить поля внутри типа `input` можно. Это еще один повод использовать агрумент `input` со вложенностью.

### 6.6. Мутация должна возвращать свой уникальный Payload-тип.

Результат выполнения мутации тоже необходимо возвращать в виде вложенного Payload-типа с уникальным именем. Это позволяет вам расширять ответы мутаций дополнительными полями с данными. При этом не ломать другие мутации, когда вы редактируете уникальный тип ответа для текущей мутации.

Даже если вы хотите вернуть только одну запись из вашей мутации, не поддавайтесь искушению вернуть этот тип напрямую. Возвращая данные напрямую (без обертки в Payload-тип), вы лишаете себя возможности в будущем легко добавить дополнительные поля для возврата. GraphQL version-less API (безверсионного апи) хорошо работает когда типы расширяются, а не модифицируются. Вложение ответа в свой кастомный Payload-тип – это инвестиции в будущие изменения вашего АПИ.

```graphql
type Mutation {
  createPerson(input: ...): CreatePersonPayload
}

type CreatePersonPayload {
  recordId: ID
  record: Person
  # ... любые другие поля, которые пожелаете
}
```

### 6.7. В ответе мутации возвращайте поле с типом Query

Если ваши мутации возвращают Payload-тип, то обязательно добавляейте в него поле `query` с типом `Query`. Это позволит клиентам за один раунд-трип не только вызвать мутацию, но и получить вагон данных для обновления своего приложения. К примеру, мы лайкаем какую-то статью `likePost` и тут же в ответе через поле `query` можем запросить любые данные которые доступны нам в АПИ (в нашем примере список последних статей с активностью `lastActivePosts`).

```graphql
mutation {
  likePost(id: 1) {
    record {
      id
      title
      likes
    }
    query {
      lastActivePosts {
        id
        title
        likes
      }
    }
  }
}
```

Если мутация возвращает `query`, то для фронтендеров открывается дичайший профит – возможность сразу запросить любые данные для своего приложения после какой-нибудь страшной мутации за один http-запрос. А если на клиенте используются Relay или ApolloClient с именованными фрагментами, то обновить половину приложение становиться проще простого. Не надо писать второй запрос на получение данных и как-то пробрасывать их в нужное место. Всё обновиться магическим образом само, просто надо написать такую мутацию с существующими фрагментами из вашего приложения:

```graphql
mutation {
  likePost(id: 1) {
    query {
      ...LastActivePostsComponent
      ...ActiveUsersComponent
    }
  }
}
```

На стороне сервера прикрутить `query` в Payload-типе можно следующим образом:

Стандартная имплементация с пакетом `graphql`:

```js
const QueryType = new GraphQLObjectType({ name: 'Query', fields: ... });
const MutationType new GraphQLObjectType({ name: 'Mutation', fields: () => {
  likePost: {
    args: { id: { type: new GraphQLNonNull(GraphQLInt) } },
    type: new GraphQLObjectType({
      name: 'LikePostPayload',
      fields: {
        record: { type: PostType },
        recordId: { type: GraphQLInt },
        query: { type: new GraphQLNonNull(QueryType) }, // ✨✨✨ magic – add 'query' field with 'Query' root-type
      },
    }),
    resolve: async (_, { id }, context) => {
      const post = await context.DB.Post.find(id);
      post.like();
      return {
        record: post,
        recordId: post.id,
        query: {}, // ✨✨✨ magic - just return empty Object
      };
    },
  }
}});
```

С помощью пакета `graphql-tools`:

```js
const typeDefs = gql`
  type Query { ...}
  type Post { ... }
  type Mutation {
    likePost(id: Int!): LikePostPayload
  }
  type LikePostPayload {
    recordId: Int
    record: Post
    # ✨✨✨ magic – add 'query' field with 'Query' root-type
    query: Query!
  }
`;

const resolvers = {
  Mutation: {
    likePost: async (_, { id }, context) => {
      const post = await context.DB.Post.find(id);
      post.like();
      return {
        record: post,
        recordId: post.id,
        query: {}, // ✨✨✨ magic - just return empty Object
      };
    },
  }
};
```

С помощью `graphql-compose`:

```js
schemaComposer.Mutation.addFields({
  likePost: {
    args: { id: 'Int!' },
    type: `type LikePostPayload {
      record: Post
      recordId: Int
      # ✨✨✨ magic – add 'query' field with 'Query' root-type
      query: Query!
    }`,
    resolve: async (_, { id }, context) => {
      const post = await context.DB.Post.find(id);
      post.like();
      return {
        record: post,
        recordId: post.id,
        query: {}, // ✨✨✨ magic - just return empty Object
      };
    },
  }
}});
```

### 6.8. Мутации должны возвращать в Payload'e поле `errors` с типизированными пользовательскими ошибками.

В резолвере можно выбросить эксепшн, и тогда ошибка улетает на глобальный уровень, но так делать нельзя по следующим причинам:

- ошибки глобального уровня используются для ошибок парсинга и других серверных ошибок
- на клиентской стороне тяжело разобрать этот массив глобальных ошибок
- клиент не знает какие ошибки могут возникнуть, они не типизированы и в схеме отсутствуют.

Мутации должные возвращать пользовательские ошибки или ошибки бизнес-логики сразу в Payload'е мутации в поле `errors`. Все ошибки необходимо описать с суффиксом `Problem`. И для самой мутации завести Union-тип ошибок, где будут перечислены возможные пользовательские ошибки. Это позволит легко определять ошибки на клиентской стороне, сразу понимать что может пойти не так. И более того позволит клиенту дозапросить дополнительные мета-данные по ошибке.

Для начала необходимо создать интерфейс для ошибок и можно объявить пару глобальных ошибок. Интерфейс необходим, чтобы можно было считать текстовое сообщение в не зависимости от того, какая ошибка вернулась. А вот каждую конкретную ошибку уже можно расширить дополнительными значениями, например в ошибке `SpikeProtectionProblem` добавлено поле `wait`:

```graphql
interface ProblemInterface {
  message: String!
}

type AccessRightProblem implements ProblemInterface {
  message: String!
}

type SpikeProtectionProblem implements ProblemInterface {
  message: String!
  # Timout in seconds when the next operation will be executed without errors
  wait: Int!
}

type PostDoesNotExistsProblem implements ProblemInterface {
  message: String!
  postId: Int!
}
```

Ну а дальше можно описать нашу мутацию `likePost` с возвратом пользовательских ошибок:

```graphql
type Mutation {
  likePost(id: Int!): LikePostPayload
}

union LikePostProblems = SpikeProtectionProblem | PostDoesNotExistsProblem;

type LikePostPayload {
  recordId: Int
  # `record` is nullable! If there is an error we may return null for Post
  record: Post
  errors: [LikePostProblems!]
}
```

Благодаря union-типу `LikePostProblems` теперь через интроспекцию фронтендеры знаю какие ошибки могут вернутся при вызове мутации `likePost`. К примеру для такого запроса они для любого типа ошибки смогут считать название ошибки с поля `__typename`, а вот благодаря интерфейсу считать `message` из любого типа ошибки:

```graphql
mutation {
  likePost(id: 666) {
    errors {
      __typename
      ... on ProblemInterface {
        message
      }
    }
  }
}
```

А если клиенты умные, то можно запросить дополнительные поля по необходимым по ошибкам:

```graphql
mutation {
  likePost(id: 666) {
    recordId
    record {
      title
      likes
    }
    errors {
      __typename
      ... on ProblemInterface {
        message
      }
      ... on SpikeProtectionProblem {
        message
        wait
      }
      ... on PostDoesNotExistsProblem {
        message
        postId
      }
    }
  }
}
```

И получить ответ от сервера в таком виде:

```js
{
  data: {
    likePost: {
      errors: [
        {
          __typename: 'PostDoesNotExistsProblem',
          message: 'Post does not exists!',
          postId: 666,
        },
        {
          __typename: 'SpikeProtectionProblem',
          message: 'Spike protection! Please retry later!',
          wait: 20,
        },
      ],
      record: { likes: 0, title: 'Post 666' },
      recordId: 666,
    },
  },
}
```

## Some rules from Shopify

To get this simplified representation, I took out all scalar fields, all field names, and all nullability information. What you're left with still looks kind of like GraphQL but lets you focus on higher level of the types and their relationships.

Rule #1: Always start with a high-level view of the objects and their relationships before you deal with specific fields.

----

The one that may have stood out to you already, and is hopefully fairly obvious, is the inclusion of the CollectionMembership type in the schema. The collection memberships table is used to represent the many-to-many relationship between products and collections. Now read that last sentence again: the relationship is between products and collections; from a semantic, business domain perspective, collection memberships have nothing to do with anything. They are an implementation detail.

Rule #2: Never expose implementation details in your API design.

----

On a closely related note, a good API does not model the user interface either. The implementation and the UI can both be used for inspiration and input into your API design, but the final driver of your decisions must always be the business domain.

Even more importantly, existing REST API choices should not necessarily be copied. The design principles behind REST and GraphQL can lead to very different choices, so don't assume that what worked for your REST API is a good choice for GraphQL.

Rule #3: Design your API around the business domain, not the implementation, user-interface, or legacy APIs.

----

Rule #5: Major business-object types should always implement Node.

```graphql
interface Node {
  id: ID!
}
```

It hints to the client that this object is persisted and retrievable by the given ID, which allows the client to accurately and efficiently manage local caches and other tricks. Most of your major identifiable business objects (e.g. products, collections, etc) should implement Node.

----

Rule #8: Always use object references instead of ID fields.

Now we come to the imageId field. This field is a classic example of what happens when you try and apply REST designs to GraphQL. In REST APIs it's pretty common to include the IDs of other objects in your response as a way to link together those objects, but this is a major anti-pattern in GraphQL. Instead of providing an ID, and forcing the client to do another round-trip to get any information on the object, we should just include the object directly into the graph — that's what GraphQL is for after all. In REST APIs this pattern often isn't practical, since it inflates the size of the response significantly when the included objects are large. However, this works fine in GraphQL because every field must be explicitly queried or the server won't return it.

```graphql
type Collection implements Node {
  id: ID!
  title: String!
  imageId: ID # <-- BAD
  image: Image
}

type Image {
  id: ID!
}
```

----

Rule #12: The API should provide business logic, not just data. Complex calculations should be done on the server, in one place, not on the client, in many places.

This last point is a critical piece of design philosophy: the server should always be the single source of truth for any business logic. An API almost always exists to serve more than one client, and if each of those clients has to implement the same logic then you've effectively got code duplication, with all the extra work and room for error which that entails.

```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Bool!
}
```

Rule #13: Provide the raw data too, even when there's business logic around it.

Clients should be able to do the business logic themselves, if they have to. You can’t predict all of the logic a client is going to want

## 9. Правила версионирования

-----

Я в корне не согласен с правилом №21 от Shopify:
> Rule #21: Structure mutation inputs to reduce duplication, even if this requires relaxing requiredness constraints on certain fields.

Экономию на типах сложно оправдать. С таким подход от Shopify будет трудно изменять ваше апи в будущем.

-----

Exposing a schema element (field, argument, type, etc) should be driven by an actual need and use case. GraphQL schemas can easily be evolved by adding elements, but changing or removing them are breaking changes and much more difficult.

Rule #4: It's easier to add fields than to remove them.

-----

## Полезные ссылки

- [Shopify: Designing a GraphQL API](https://github.com/Shopify/graphql-design-tutorial/blob/master/TUTORIAL.md)
- [Designing GraphQL Mutations by Caleb Meredith](https://blog.apollographql.com/designing-graphql-mutations-e09de826ed97)