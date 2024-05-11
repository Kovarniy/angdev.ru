# Тестирование автономных компонентов

С автономными компонентами Angular становится намного легче: NgModules стали необязательными, а значит, мы можем работать с меньшим количеством косвенных зависимостей. Чтобы сделать это возможным, компоненты теперь напрямую ссылаются на свои зависимости: другие компоненты, а также директивы и пайпы. Существуют так называемые Standalone API для настройки сервисов, таких как `HttpClient`.

Дополнительные Standalone API предоставляют имитаторы для автоматизации тестирования. Здесь я собираюсь представить эти API. Для этого я сосредоточусь на встроенных инструментах, поставляемых с Angular. Использованные 📂 [примеры](https://github.com/manfredsteyer/standalone-example-cli.git) можно найти [здесь](https://github.com/manfredsteyer/standalone-example-cli.git).

Если вы не хотите использовать только штатные ресурсы, то в ветке `third-party-testing` вы найдете те же примеры, основанные на новом Cypress Component Test Runner и на Testing Library.

## Настройка тестов {#leanpub-auto-test-setup}

Несмотря на то, что в Standalone Components модули необязательны, `TestBed` все равно поставляется с модулем тестирования. Он заботится о настройке теста и предоставляет все компоненты, директивы, пайпы и сервисы для теста:

```ts
import { provideHttpClient } from '@angular/common/http';
import {
    HttpTestingController,
    provideHttpClientTesting,
} from '@angular/common/http/testing';

/* […] */

describe('FlightSearchComponent', () => {
    let component: FlightSearchComponent;
    let fixture: ComponentFixture<FlightSearchComponent>;
    beforeEach(async () => {
        await TestBed.configureTestingModule({
            imports: [FlightSearchComponent],
            providers: [
                provideHttpClient(),
                provideHttpClientTesting(),

                provideRouter([]),

                provideStore(),
                provideState(bookingFeature),
                provideEffects(BookingEffects),
            ],
        }).compileComponents();

        fixture = TestBed.createComponent(
            FlightSearchComponent
        );
        component = fixture.componentInstance;
        fixture.detectChanges();
    });

    it('should search for flights', () => {
        /* […] */
    });
});
```

В приведенном примере импортируется тестируемый автономный компонент и предоставляются необходимые сервисы через массив `providers`. Именно здесь и вступают в игру упомянутые автономные API. Они предоставляют услуги для `HttpClient`, маршрутизатора и NGRX.

Функция `provideStore` устанавливает хранилище NGRX, `provideState` предоставляет фрагмент функции, необходимый для теста, а `provideEffects` регистрирует связанный эффект. Ниже мы заменим эти конструкции на макеты.

Интересен метод `provideHttpClientTesting`: он заменяет `HttpBackend`, используемый за кулисами `HttpClient`, на `HttpTestingBackend`, который имитирует HTTP-вызовы. Следует отметить, что его необходимо вызывать после (!) `provideHttpClient`.

Поэтому сначала необходимо настроить `HttpClient` по умолчанию, чтобы затем переписать отдельные детали для тестирования. Эту схему мы еще увидим ниже при тестировании маршрутизатора.

## Макет HttpClient {#leanpub-auto-the-httpclient-mock}

После настройки `HttpClient` и `HttpTestingBackend` отдельные тесты реализуются как обычно: тест использует `HttpTestingController` для получения информации об ожидающих HTTP-запросах и указания симулируемых HTTP-ответов:

```ts
it('should search for flights', () => {
    component.from = 'Paris';
    component.to = 'London';
    component.search();

    const ctrl = TestBed.inject(HttpTestingController);

    const req = ctrl.expectOne(
        'https://[…]/flight?from=Paris&amp;to=London'
    );
    req.flush([{}, {}, {}]); // return 3 empty objects as dummy flights

    component.flights$.subscribe((flights) => {
        expect(flights.length).toBe(3);
    });

    ctrl.verify();
});
```

Затем тест проверяет, обработал ли компонент смоделированный HTTP-ответ так, как было задумано. В показанном случае тест предполагает, что компонент предлагает полученные рейсы через свое свойство `flights`.

В конце теста проверяется, что больше нет HTTP-запросов, на которые еще не был получен ответ. Для этого он вызывает метод `verify`, предоставляемый `HttpTestingController`. Если в этот момент все еще есть открытые запросы, `verify` выбрасывает исключение, которое приводит к неудаче теста.

## Неглубокое тестирование {#leanpub-auto-shallow-testing}

Если вы тестируете компонент, то автоматически тестируются и все подкомпоненты, директивы и пайпы, используемые в шаблоне. Это нежелательно, особенно для модульных тестов, которые фокусируются на одной единице кода. Кроме того, такое поведение замедляет выполнение тестов при наличии большого количества зависимостей.

Для предотвращения этого используются неглубокие тесты. Это означает, что при настройке тестов все зависимости заменяются имитаторами. Эти имитаторы должны иметь тот же интерфейс, что и заменяемые зависимости. В случае компонентов это означает, помимо прочего, что должны быть предложены те же свойства и события (входы и выходы), а также должны использоваться те же селекторы.

Для замены этих зависимостей в `TestBed` предлагается метод `overrideComponent`:

```ts
await TestBed.configureTestingModule(/* […] */)
    .overrideComponent(FlightSearchComponent, {
        remove: { imports: [FlightCardComponent] },
        add: { imports: [FlightCardMock] },
    })
    .compileComponents();
```

В показанном случае `FlightSearchComponent` использует в своем шаблоне другой автономный компонент: `FlightCardComponent`. Технически это означает, что `FlightCardComponent` появляется в массиве `imports` компонента `FlightSearchComponent`. Для реализации неглубокого теста эта запись удаляется. В качестве замены добавляется `FlightCardMock`. Об этом позаботятся методы `remove` и `add`.

Таким образом, `FlightSearchComponent` используется в тесте без реальных зависимостей. Тем не менее, тест может проверить, ведут ли компоненты себя так, как нужно. Например, в следующем листинге проверяется, создает ли `FlightSearchComponent` элемент с именем `flight-card` для каждого найденного рейса.

```ts
it('should display a flight-card for each found flight', () => {
    component.from = 'Paris';
    component.to = 'London';
    component.search();

    const ctrl = TestBed.inject(HttpTestingController);

    const req = ctrl.expectOne(
        'https://[…]/flight?from=Paris&amp;to=London'
    );
    req.flush([{}, {}, {}]);

    fixture.detectChanges();

    const cards = fixture.debugElement.queryAll(
        By.css('flight-card')
    );
    expect(cards.length).toBe(3);
});
```

## Имитация маршрутизатора и магазина {#leanpub-auto-mock-router-and-store}

Тестовая установка, использованная до сих пор, имитировала только `HttpCient`. Однако существуют также автономные API для имитации маршрутизатора и NGRX:

```ts
import { provideRouter } from '@angular/router';
import { provideLocationMocks } from '@angular/common/testing';

import { provideMockStore } from '@ngrx/store/testing';
import { provideMockActions } from '@ngrx/effects/testing';

/* […] */

describe('FlightSearchComponent (at router level)', () => {
    let component: FlightSearchComponent;
    let fixture: ComponentFixture<FlightSearchComponent>;
    let actions$ = new Subject<Action>();

    beforeEach(async () => {
        await TestBed.configureTestingModule({
            providers: [
                provideHttpClient(),
                provideHttpClientTesting(),

                provideRouter([
                    {
                        path: 'flight-edit/:id',
                        component: FlightEditComponent,
                    },
                ]),
                provideLocationMocks(),

                provideMockStore({
                    initialState: {
                        [BOOKING_FEATURE_KEY]: {
                            flights: [
                                { id: 1 },
                                { id: 2 },
                                { id: 3 },
                            ],
                        },
                    },
                }),

                provideMockActions(() => actions$),
            ],
            imports: [FlightSearchComponent],
        }).compileComponents();

        fixture = TestBed.createComponent(
            FlightSearchComponent
        );
        component = fixture.componentInstance;
        fixture.detectChanges();
    });

    /* […] */
});
```

Как и при тестировании `HttpClient`, тест сначала настраивает маршрутизатор обычным способом. Затем он использует `provideLocationMocks` для переопределения пары внутренних сервисов, а именно `Location` и `LocationStrategy`. Эта процедура позволяет смоделировать изменение маршрута в тестовых примерах. Вместо традиционного магазина используется `MockStore`, который также поставляется с NGRX. Он позволяет свободно определять все содержимое магазина. Это делается либо вызовом `provideMockStore`, либо через его метод `setState`. Кроме того, `provideMockActions` дает нам возможность заменить наблюдаемую `actions$`, на которую часто полагаются эффекты NGRX. Тестовый пример, использующий эту настройку, может выглядеть следующим образом:

```ts
it('routes to flight-card', fakeAsync(() => {
    const link = fixture.debugElement.query(
        By.css('a[class*=btn-default ]')
    );
    link.nativeElement.click();

    flush();
    fixture.detectChanges();

    const location = TestBed.inject(Location);
    expect(location.path()).toBe(
        '/flight-edit/1;showDetails=false'
    );
}));
```

Этот тест предполагает, что `FlightSearchComponent` отображает по одной ссылке на каждый рейс в (макетном) магазине. Он имитирует нажатие на первую ссылку и проверяет, переключится ли приложение на ожидаемый маршрут. Чтобы Angular обработал имитацию щелчка и запустил изменение маршрута, должно быть запущено обнаружение изменений. К сожалению, это не происходит автоматически в тестах. Вместо этого его нужно запускать с помощью метода `detectChanges`, когда это необходимо. Задействованные операции являются асинхронными. Поэтому используется `fakeAsync`, чтобы мы не обременяли себя этим. Это позволяет синхронно обрабатывать отложенные микрозадачи с помощью `flush.`.

## Эффекты тестирования

В `MockStore` не запускаются редукторы или эффекты. Первые являются просто функциями и могут быть протестированы простым способом. Замена `action$` — хороший способ протестировать эффекты. Настройка теста в предыдущем разделе уже позаботилась об этом. Теперь тест, основанный на ней, может использовать наблюдаемую `action$` для отправки действия, на которое реагирует тестируемый эффект:

```ts
it('load flights', () => {
    const effects = TestBed.inject(BookingEffects);
    let flights: Flight[] = [];

    effects.loadFlights$.subscribe((action) => {
        flights = action.flights; // Action returned from Effect
    });

    actions$.next(
        loadFlights({ from: 'Paris', to: 'London' })
    );
    // Action sent to store to invoke Effect

    const ctrl = TestBed.inject(HttpTestingController);
    const req = ctrl.expectOne(
        'https://[…]/flight?from=Paris&amp;to=London'
    );
    req.flush([{}, {}, {}]);

    expect(flights.length).toBe(3);
});
```

В рассматриваемом случае эффект вызывает HTTP-вызов, на который отвечает `HttpTestingController`. Ответ содержит три рейса, для простоты представленные тремя пустыми объектами. Наконец, тест проверяет, предоставил ли эффект эти рейсы через исходящее действие.

## Заключение {#leanpub-auto-conclusion-5}

Все больше библиотек предлагают автономные API для подражания зависимостям. Они либо предоставляют имитирующую реализацию, либо, по крайней мере, перезаписывают сервисы в реальной реализации, чтобы повысить тестируемость. Модуль `TestingModule` по-прежнему используется для создания тестовых настроек. Однако, в отличие от предыдущего, теперь он импортирует отдельные компоненты, директивы и пайпы, которые будут тестироваться. Их классические аналоги, напротив, объявляются. Кроме того, `TestingModule` теперь включает в себя провайдеры, устанавливаемые Standalone API.
