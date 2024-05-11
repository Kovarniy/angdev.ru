# Паттерны для автономных API

Вместе с автономными компонентами команда Angular представила автономные API. Они позволяют настраивать библиотеки более легким способом. Примерами библиотек, предоставляющих Standalone API, на данный момент являются `HttpClient` и `Router`. Также ранним последователем этой идеи является NGRX.

В этой главе я представляю несколько паттернов для написания собственных Standalone API, взятых из вышеупомянутых библиотек. Для каждого паттерна обсуждаются следующие аспекты: намерения, лежащие в основе паттерна, описание, пример реализации, примеры, встречающиеся в упомянутых библиотеках, и вариации деталей реализации.

Большинство из этих паттернов особенно интересны для авторов библиотек. Они способны улучшить DX для потребителей библиотеки. С другой стороны, большинство из них могут оказаться излишними для приложений.

> Большое спасибо [Алексу Рикабо](https://twitter.com/synalx) из Angular за вычитку и обратную связь!

📂 [Исходный код, использованный в примерах](https://github.com/manfredsteyer/standalone-example-cli.git)

## Пример для паттернов {#leanpub-auto-case-study-for-patterns}

Для представления выведенных закономерностей используется простая библиотека логгеров. Эта библиотека настолько проста, насколько это возможно, но настолько сложна, насколько это необходимо для демонстрации реализации паттернов.

Каждое сообщение журнала имеет `LogLevel`, определяемый перечислением:

```ts
export enum LogLevel {
    DEBUG = 0,
    INFO = 1,
    ERROR = 2,
}
```

Для простоты мы ограничим нашу библиотеку Logger только тремя уровнями журнала.

Абстрактный `LoggerConfig` определяет возможные параметры конфигурации:

```ts
export abstract class LoggerConfig {
    abstract level: LogLevel;
    abstract formatter: Type<LogFormatter>;
    abstract appenders: Type<LogAppender>[];
}
```

Это абстрактный класс, так как интерфейсы не могут быть использованы в качестве маркеров для DI. Константа этого класса определяет значения по умолчанию для параметров конфигурации:

```ts
export const defaultConfig: LoggerConfig = {
    level: LogLevel.DEBUG,
    formatter: DefaultLogFormatter,
    appenders: [DefaultLogAppender],
};
```

Форматтер `LogFormatter` используется для форматирования сообщений журнала перед их публикацией через `LogAppender`:

```ts
export abstract class LogFormatter {
    abstract format(
        level: LogLevel,
        category: string,
        msg: string
    ): string;
}
```

Как и `LoggerConfiguration`, `LogFormatter` — это абстрактный класс, используемый в качестве маркера. Потребитель библиотеки логгеров может настроить форматирование, предоставив свою собственную реализацию. В качестве альтернативы они могут использовать реализацию по умолчанию, предоставляемую библиотекой:

```ts
@Injectable()
export class DefaultLogFormatter implements LogFormatter {
    format(
        level: LogLevel,
        category: string,
        msg: string
    ): string {
        const levelString = LogLevel[level].padEnd(5);
        return `[${levelString}] ${category.toUpperCase()} ${msg}`;
    }
}
```

`LogAppender` — это еще одна сменная концепция, отвечающая за добавление сообщения журнала в журнал:

```ts
export abstract class LogAppender {
    abstract append(
        level: LogLevel,
        category: string,
        msg: string
    ): void;
}
```

Реализация по умолчанию записывает сообщение в консоль:

```ts
@Injectable()
export class DefaultLogAppender implements LogAppender {
    append(
        level: LogLevel,
        category: string,
        msg: string
    ): void {
        console.log(category + ' ' + msg);
    }
}
```

Хотя может быть только один `LogFormatter`, библиотека поддерживает несколько `LogAppender`. Например, первый `LogAppender` может записывать сообщение в консоль, а второй — отправлять его на сервер.

Чтобы сделать это возможным, отдельные `LogAppender`ы регистрируются через мультипровайдеров. Поэтому инжектор возвращает их все в массиве. Поскольку массив не может быть использован в качестве DI-токена, в примере вместо него используется `InjectionToken`:

```ts
export const LOG_APPENDERS = new InjectionToken<
    LogAppender[]
>('LOG_APPENDERS');
```

Сам `LoggserService` получает `LoggerConfig`, `LogFormatter` и массив с `LogAppenders` через DI и позволяет регистрировать сообщения для нескольких `LogLevels`:

```ts
@Injectable()
export class LoggerService {
    private config = inject(LoggerConfig);
    private formatter = inject(LogFormatter);
    private appenders = inject(LOG_APPENDERS);

    log(
        level: LogLevel,
        category: string,
        msg: string
    ): void {
        if (level < this.config.level) {
            return;
        }
        const formatted = this.formatter.format(
            level,
            category,
            msg
        );
        for (const a of this.appenders) {
            a.append(level, category, formatted);
        }
    }

    error(category: string, msg: string): void {
        this.log(LogLevel.ERROR, category, msg);
    }

    info(category: string, msg: string): void {
        this.log(LogLevel.INFO, category, msg);
    }

    debug(category: string, msg: string): void {
        this.log(LogLevel.DEBUG, category, msg);
    }
}
```

## Золотое правило {#leanpub-auto-the-golden-rule}

Прежде чем приступить к представлению выведенных паттернов, я хочу подчеркнуть то, что я называю золотым правилом предоставления сервисов:

> Всегда, когда это возможно, используйте `@Injectable({providedIn: 'root'})`!

Особенно в коде приложений, но в некоторых ситуациях и в библиотеках, это то, что вы хотите иметь: Это легко, древовидно и даже работает с ленивой загрузкой. Последний аспект — заслуга не столько Angular, сколько лежащего в его основе бандлера: Все, что просто необходимо в ленивом бандле, помещено туда.

## Паттерн: Фабрика провайдеров {#leanpub-auto-pattern-provider-factory}

**Намерения**

-   Предоставление сервисов для многократно используемой библиотеки
-   Конфигурирование многократно используемой библиотеки
-   Обмен определенными деталями реализации

**Описание**

Фабрика провайдеров — это функция, возвращающая массив с провайдерами для заданной библиотеки. Этот массив преобразуется в тип Angular `EnvironmentProviders`, чтобы убедиться, что провайдеры могут быть использованы только в области видимости окружения — в первую очередь, в корневой области видимости и области видимости, введенной с помощью конфигураций ленивой маршрутизации.

Angular и NGRX размещают такие функции в файлах под названием `provider.ts`.

**Пример**

Следующая функция провайдера `provideLogger` принимает частичную конфигурацию `LoggerConfiguration` и использует ее для создания некоторых провайдеров:

```ts
export function provideLogger(
    config: Partial<LoggerConfig>
): EnvironmentProviders {
    // using default values for missing properties
    const merged = { ...defaultConfig, ...config };

    return makeEnvironmentProviders([
        {
            provide: LoggerConfig,
            useValue: merged,
        },
        {
            provide: LogFormatter,
            useClass: merged.formatter,
        },
        merged.appenders.map((a) => ({
            provide: LOG_APPENDERS,
            useClass: a,
            multi: true,
        })),
    ]);
}
```

Недостающие значения конфигурации берутся из конфигурации по умолчанию. Angular's `makeEnvironmentProviders` оборачивает массив `Provider` в экземпляр `EnvironmentProviders`.

Эта функция позволяет потребляющему приложению настроить логгер во время загрузки, как и другие библиотеки, например, `HttpClient` или `Router`:

```ts
bootstrapApplication(AppComponent, {
    providers: [
        provideHttpClient(),

        provideRouter(APP_ROUTES),

        /* [...] */

        // Setting up the Logger:
        provideLogger(loggerConfig),
    ],
});
```

**Случаи и вариации**

-   Это обычный паттерн, используемый во всех рассмотренных библиотеках.
-   Фабрики провайдеров для `Router` и `HttpClient` имеют второй необязательный параметр, который принимает дополнительные возможности (см. паттерн _Feature_, ниже).
-   Вместо передачи конкретной реализации сервиса, например, LogFormatter, NGRX позволяет принимать либо токен, либо конкретный объект для редукторов.
-   В `HttpClient` передается массив с функциональными перехватчиками через функцию `with` (см. паттерн _Feature_, ниже). Эти функции также регистрируются как сервисы.

## Паттерн: Функция {#leanpub-auto-pattern-feature}

**Намерения**

-   Активация и настройка дополнительных функций
-   Сделать эти функции изменяемыми в дереве
-   Предоставление базовых сервисов через текущее окружение

**Описание**

Фабрика провайдеров принимает необязательный массив с объектом функции. Каждый объект функции имеет идентификатор `kind` и массив `providers`. Свойство `kind` позволяет проверить комбинацию передаваемых функций. Например, могут быть взаимоисключающие функции, такие как настройка обработки токенов XSRF и отключение обработки токенов XSRF для `HttpClient`.

**Пример**

В нашем примере используется функция цвета, которая позволяет отображать сообщения разных `LoggerLevel` разными цветами:

Для категоризации функций используется перечисление:

```ts
export enum LoggerFeatureKind {
    COLOR,
    OTHER_FEATURE,
    ADDITIONAL_FEATURE,
}
```

Каждый признак представлен объектом `LoggerFeature`:

```ts
export interface LoggerFeature {
    kind: LoggerFeatureKind;
    providers: Provider[];
}
```

Для предоставления цветовой характеристики вводится фабричная функция, следующая шаблону именования _Feature_:

```ts
export function withColor(
    config?: Partial<ColorConfig>
): LoggerFeature {
    const internal = { ...defaultColorConfig, ...config };

    return {
        kind: LoggerFeatureKind.COLOR,
        providers: [
            {
                provide: ColorConfig,
                useValue: internal,
            },
            {
                provide: ColorService,
                useClass: DefaultColorService,
            },
        ],
    };
}
```

Фабрика провайдеров принимает несколько функций через необязательный второй параметр, заданный в виде массива rest:

```ts
export function provideLogger(
    config: Partial<LoggerConfig>,
    ...features: LoggerFeature[]
): EnvironmentProviders {
    const merged = { ...defaultConfig, ...config };

    // Inspecting passed features
    const colorFeatures =
        features?.filter(
            (f) => f.kind === LoggerFeatureKind.COLOR
        )?.length ?? 0;

    // Validating passed features
    if (colorFeatures > 1) {
        throw new Error(
            'Only one color feature allowed for logger!'
        );
    }

    return makeEnvironmentProviders([
        {
            provide: LoggerConfig,
            useValue: merged,
        },
        {
            provide: LogFormatter,
            useClass: merged.formatter,
        },
        merged.appenders.map((a) => ({
            provide: LOG_APPENDERS,
            useClass: a,
            multi: true,
        })),

        // Providing services for the features
        features?.map((f) => f.providers),
    ]);
}
```

Свойство `kind` функции используется для проверки и подтверждения переданных функций. Если все в порядке, провайдеры, найденные в характеристике, помещаются в возвращаемый объект `EnvironmentProviders`.

В результате инъекции зависимостей `DefaultLogAppender` получает `ColorService`, предоставляемый функцией цвета:

```ts
export class DefaultLogAppender implements LogAppender {
    colorService = inject(ColorService, { optional: true });

    append(
        level: LogLevel,
        category: string,
        msg: string
    ): void {
        if (this.colorService) {
            msg = this.colorService.apply(level, msg);
        }
        console.log(msg);
    }
}
```

Поскольку функции являются необязательными, `DefaultLogAppender` передает `optional: true` в `inject`. В противном случае мы получили бы исключение, если бы функция не была применена. Кроме того, `DefaultLogAppender` должен проверять значения `null`.

**Случаи и вариации**

-   Маршрутизатор `Router` использует его, например, для настройки предварительной загрузки или для активации трассировки отладки.
-   `HttpClient` использует его, например, для предоставления перехватчиков, настройки JSONP и настройки/отключения обработки токенов XSRF.
-   И `Router`, и `HttpClient` объединяют возможные возможности в союзный тип (например, `export type AllowedFeatures = ThisFeature | ThatFeature`). Это помогает IDE предлагать встроенные функции.
-   Некоторые реализации инжектируют текущий `Injector` и используют его, чтобы узнать, какие функции были настроены. Это императивная альтернатива использованию `optional: true`.
-   В реализациях функций Angular свойства `kind` и `providers` префиксируются `ɵ` и, следовательно, объявляются как внутренние свойства.

## Паттерн: Фабрика поставщиков конфигурации {#leanpub-auto-pattern-configuration-provider-factory}

**Намерения**

-   Конфигурирование существующих сервисов
-   Предоставление дополнительных сервисов и их регистрация в существующих сервисах
-   Расширение поведения сервиса из вложенного окружения.

**Описание**

Фабрики поставщиков конфигурации расширяют поведение существующей службы. Они могут предоставлять дополнительные сервисы и использовать `ENVIRONMENT_INITIALIZER` для получения экземпляров как предоставляемых сервисов, так и существующих сервисов для расширения.

**Пример**

Предположим, что расширенная версия нашего `LoggerService` позволяет определять дополнительный `LogAppender` для каждой категории журналов:

```ts
@Injectable()
export class LoggerService {
    private appenders = inject(LOG_APPENDERS);
    private formatter = inject(LogFormatter);
    private config = inject(LoggerConfig);
    /* [...] */

    // Additional LogAppender per log category
    readonly categories: Record<string, LogAppender> = {};

    log(
        level: LogLevel,
        category: string,
        msg: string
    ): void {
        if (level < this.config.level) {
            return;
        }

        const formatted = this.formatter.format(
            level,
            category,
            msg
        );

        // Lookup appender for this very category and use
        // it, if there is one:
        const catAppender = this.categories[category];

        if (catAppender) {
            catAppender.append(level, category, formatted);
        }

        // Also, use default appenders:
        for (const a of this.appenders) {
            a.append(level, category, formatted);
        }
    }

    /* [...] */
}
```

Чтобы сконфигурировать `LogAppender` для категории, мы можем ввести еще одну фабрику провайдеров:

```ts
export function provideCategory(
    category: string,
    appender: Type<LogAppender>
): EnvironmentProviders {
    // Internal/ Local token for registering the service
    // and retrieving the resolved service instance
    // immediately after.
    const appenderToken = new InjectionToken<LogAppender>(
        'APPENDER_' + category
    );

    return makeEnvironmentProviders([
        {
            provide: appenderToken,
            useClass: appender,
        },
        {
            provide: ENVIRONMENT_INITIALIZER,
            multi: true,
            useValue: () => {
                const appender = inject(appenderToken);
                const logger = inject(LoggerService);

                logger.categories[category] = appender;
            },
        },
    ]);
}
```

Эта фабрика создает провайдера для класса `LogAppender`. Однако нам нужен не сам класс, а его экземпляр. Также нам нужен `Injector` для разрешения зависимостей этого экземпляра. И то, и другое происходит при получении `LogAppender` через инжектор.

Именно это и делает `ENVIRONMENT_INITIALIZER`, который является мультипровайдером, привязанным к токену `ENVIRONMENT_INITIALIZER` и указывающим на функцию. Она получает инжектированный `LogAppender`, а также `LoggerService`. Затем `LogAppender` регистрируется в логгере.

Это позволяет расширить существующий `LoggerService`, который может даже исходить из родительской области видимости. Например, в следующем примере предполагается, что `LoggerService` находится в корневой области видимости, а дополнительная категория журнала устанавливается в области видимости ленивого маршрута:

```ts
export const FLIGHT_BOOKING_ROUTES: Routes = [
    {
        path: '',
        component: FlightBookingComponent,

        // Providers for this route and child routes
        // Using the providers array sets up a new
        // environment injector for this part of the
        // application.
        providers: [
            // Setting up an NGRX feature slice
            provideState(bookingFeature),
            provideEffects([BookingEffects]),

            // Provide LogAppender for logger category
            provideCategory('booking', DefaultLogAppender),
        ],
        children: [
            {
                path: 'flight-search',
                component: FlightSearchComponent,
            },
            /* [...] */
        ],
    },
];
```

**Случаи и вариации**

-   `@ngrx/store` использует этот паттерн для регистрации фрагментов функций
-   `@ngrx/effects` использует этот паттерн для подключения эффектов, предоставляемых функцией.
-   Функция `withDebugTracing` использует этот паттерн, чтобы подписаться на наблюдаемую `events` маршрутизатора.

## Паттерн: NgModule Bridge {#leanpub-auto-pattern-ngmodule-bridge}

**Намерения**

-   Не ломать существующий код, использующий `NgModules` при переходе на Standalone API.
-   Позволяет таким частям приложения устанавливать `EnvironmentProviders`, которые приходят из Provider Factory.

Замечания: Для нового кода этот паттерн кажется излишним, поскольку фабрика провайдеров может быть вызвана напрямую для потребляющих (унаследованных) NgModules.

**Описание**

Мост NgModule Bridge — это NgModule, получающий (некоторые) свои провайдеры через фабрику провайдеров (см. паттерн _Фабрика провайдеров_). Чтобы дать вызывающему модулю больше контроля над предоставляемыми услугами, можно использовать статические методы типа `forRoot`. Эти методы могут принимать объект конфигурации.

**Пример**

Следующий `NgModules` позволяет настроить логгер традиционным способом:

```ts
@NgModule({
    imports: [
        /* your imports here */
    ],
    exports: [
        /* your exports here */
    ],
    declarations: [
        /* your delarations here */
    ],
    providers: [
        /* providers, you _always_ want to get, here */
    ],
})
export class LoggerModule {
    static forRoot(
        config = defaultConfig
    ): ModuleWithProviders<LoggerModule> {
        return {
            ngModule: LoggerModule,
            providers: [provideLogger(config)],
        };
    }

    static forCategory(
        category: string,
        appender: Type<LogAppender>
    ): ModuleWithProviders<LoggerModule> {
        return {
            ngModule: LoggerModule,
            providers: [
                provideCategory(category, appender),
            ],
        };
    }
}
```

Чтобы избежать повторной реализации фабрик провайдеров, методы модуля делегируются им. Поскольку использование таких методов является обычным делом при работе с NgModules, потребители могут использовать существующие знания и соглашения.

**Случаи и вариации**

-   Все рассмотренные библиотеки используют этот паттерн для сохранения обратной совместимости

## Паттерн: Цепочка сервисов {#leanpub-auto-pattern-service-chain}

**Намерения**

-   Делегирование сервиса другому своему экземпляру в родительской области видимости.

**Описание**

Когда один и тот же сервис размещается в нескольких вложенных инжекторах окружения, мы обычно получаем только экземпляр сервиса в текущей области видимости. Таким образом, вызов сервиса во вложенной области видимости не будет выполняться в родительской области видимости. Чтобы обойти эту проблему, сервис может найти экземпляр самого себя в родительской области видимости и делегировать его.

**Пример**

Предположим, что мы снова предоставляем библиотеку logger для ленивого маршрута:

```ts
export const FLIGHT_BOOKING_ROUTES: Routes = [
    {
        path: '',
        component: FlightBookingComponent,
        canActivate: [
            () => inject(AuthService).isAuthenticated(),
        ],
        providers: [
            // NGRX
            provideState(bookingFeature),
            provideEffects([BookingEffects]),

            // Providing **another** logger for this part of the app:
            provideLogger(
                {
                    level: LogLevel.DEBUG,
                    chaining: true,
                    appenders: [DefaultLogAppender],
                },
                withColor({
                    debug: 42,
                    error: 43,
                    info: 46,
                })
            ),
        ],
        children: [
            {
                path: 'flight-search',
                component: FlightSearchComponent,
            },
            /* [...] */
        ],
    },
];
```

Это устанавливает **другой** набор сервисов Logger в инжекторе окружения этого ленивого маршрута и его дочерних элементов. Эти сервисы являются тенью своих аналогов в корневой области видимости. Таким образом, когда компонент в ленивой области видимости вызывает `LoggerService`, сервисы в корневой области видимости не срабатывают.

Чтобы предотвратить это, мы можем получить `LoggerService` из родительской области видимости. Точнее, это не _родительский_ scope, а "ближайший предковый scope", предоставляющий `LoggerService`. После этого служба может делегировать полномочия своему родителю. Таким образом, сервисы связываются в цепочку:

```ts
@Injectable()
export class LoggerService {
    private appenders = inject(LOG_APPENDERS);
    private formatter = inject(LogFormatter);
    private config = inject(LoggerConfig);

    private parentLogger = inject(LoggerService, {
        optional: true,
        skipSelf: true,
    });
    /* [...] */

    log(
        level: LogLevel,
        category: string,
        msg: string
    ): void {
        // 1. Делайте здесь свои дела
        /* [...] */

        // 2. Передать родителям
        if (this.config.chaining && this.parentLogger) {
            this.parentLogger.log(level, category, msg);
        }
    }
    /* [...] */
}
```

При использовании inject для получения родительского `LoggerService`, нам нужно передать `optional: true`, чтобы избежать исключения, если не найдется ни одной области-предка с `LoggerService`. Передача `skipSelf: true` гарантирует, что поиск будет производиться только в областях-предшественниках. В противном случае Angular начнет поиск с текущего диапазона и получит вызывающий сервис самостоятельно.

Кроме того, пример, показанный здесь, позволяет активировать/деактивировать это поведение с помощью нового флага `chaining` в `LoggerConfiguration`.

**Случаи и вариации**

-   В `HttpClient` этот паттерн используется также для запуска `HttpInterceptors` в родительских диапазонах. Более подробно о [цепочке HttpInterceptors](https://www.angulararchitects.io/aktuelles/the-refurbished-httpclient-in-angular-15-standalone-apis-and-functional-interceptors/) можно прочитать [здесь](https://www.angulararchitects.io/aktuelles/the-refurbished-httpclient-in-angular-15-standalone-apis-and-functional-interceptors/). Здесь поведение цепочки может быть активировано с помощью отдельной функции. Технически, эта функция регистрирует другой перехватчик, делегирующий функции в родительской области видимости.

## Паттерн: Функциональный сервис {#leanpub-auto-pattern-functional-service}

**Намерения**

-   Сделать использование библиотек более легким за счет использования функций в качестве сервисов
-   Сокращение непрямых связей за счет использования специальных функций.

**Описание**

Вместо того, чтобы заставлять потребителя реализовывать сервис на основе класса, следуя заданному интерфейсу, библиотека также принимает функции. Внутри библиотеки они могут быть зарегистрированы как сервис с помощью `useValue`.

**Пример**

В этом примере потребитель может напрямую передать функцию, действующую как `LogFormatter`, в `provideLogger`:

```ts
bootstrapApplication(AppComponent, {
    providers: [
        provideLogger(
            {
                level: LogLevel.DEBUG,
                appenders: [DefaultLogAppender],

                // Functional CSV-Formatter
                formatter: (level, cat, msg) =>
                    [level, cat, msg].join(';'),
            },
            withColor({
                debug: 3,
            })
        ),
    ],
});
```

Для этого в логгере используется тип `LogFormatFn`, определяющий сигнатуру функции:

```ts
export type LogFormatFn = (
    level: LogLevel,
    category: string,
    msg: string
) => string;
```

Также, поскольку функции не могут использоваться в качестве токенов, вводится `InjectionToken`:

```ts
export const LOG_FORMATTER = new InjectionToken<
    LogFormatter | LogFormatFn
>('LOG_FORMATTER');
```

Этот `InjectionToken` поддерживает как основанные на классах `LogFormatter`, так и функциональные. Это позволяет не ломать существующий код. Как следствие поддержки обоих вариантов, `provideLogger` должен обрабатывать оба случая немного по-разному:

```ts
export function provideLogger(
    config: Partial<LoggerConfig>,
    ...features: LoggerFeature[]
): EnvironmentProviders {
    const merged = { ...defaultConfig, ...config };

    /* [...] */

    return makeEnvironmentProviders([
        LoggerService,
        {
            provide: LoggerConfig,
            useValue: merged,
        },

        // Register LogFormatter
        //  - Functional LogFormatter:  useValue
        //  - Class-based LogFormatters: useClass
        typeof merged.formatter === 'function'
            ? {
                  provide: LOG_FORMATTER,
                  useValue: merged.formatter,
              }
            : {
                  provide: LOG_FORMATTER,
                  useClass: merged.formatter,
              },

        merged.appenders.map((a) => ({
            provide: LOG_APPENDERS,
            useClass: a,
            multi: true,
        })),
        /* [...] */
    ]);
}
```

В то время как сервисы, основанные на классах, регистрируются с помощью `useClass`, для их функциональных аналогов подходит `useValue`.

Кроме того, потребители `LogFormatter` должны быть готовы как к функциональному, так и к классовому подходу:

```ts
@Injectable()
export class LoggerService {
    private appenders = inject(LOG_APPENDERS);
    private formatter = inject(LOG_FORMATTER);
    private config = inject(LoggerConfig);

    /* [...] */

    private format(
        level: LogLevel,
        category: string,
        msg: string
    ): string {
        if (typeof this.formatter === 'function') {
            return this.formatter(level, category, msg);
        } else {
            return this.formatter.format(
                level,
                category,
                msg
            );
        }
    }

    log(
        level: LogLevel,
        category: string,
        msg: string
    ): void {
        if (level < this.config.level) {
            return;
        }

        const formatted = this.format(level, category, msg);

        /* [...] */
    }
    /* [...] */
}
```

**Случаи и вариации**

-   `HttpClient` позволяет использовать функциональные перехватчики. Они регистрируются через функцию (см. паттерн _Feature_).
-   Маршрутизатор `Router` позволяет использовать функции для реализации охранников и резолверов.

## Заключение {#leanpub-auto-conclusion-6}

Фабрики провайдеров — это простые функции, возвращающие массив с провайдерами. Они используются для получения всех провайдеров, необходимых для настройки подсистемы или библиотеки. По условию, такие фабрики имеют шаблон именования `privateXY`.

Фабрика провайдеров может принимать объект конфигурации и необязательные функции. Опциональная функция — это другая функция, возвращающая все провайдеры, необходимые для данной функции. Их имена следуют шаблону именования `withXYZ`.

Для подключения сервисов можно использовать `ENVIRONMENT_INITIALIZER`, а использование inject вместе с такими параметрами, как `optional` и `skipSelf`, позволяет создать цепочку с другим экземпляром того же сервиса в родительской области видимости.
