# Billennium-workshops-03-State-management-and-application-flow/architecture

Warsztaty wprowadzające w różne techniki zarządzania stanem i architekture aplikacji frontendowej.

> Important: Jak wyżej napisane "wprowadzenie", aby temat bardziej zgłębić i zrozumieć potrzebna jest praca własna z materiałem.

### Czym jest state / state management - na przykładzie Jasia

- Dla uproszczenia przyjmijmy, że rozmawiamy na temat matury z Jasiem.
- Tłumaczymy mu jakieś skomplikowane zagadnienie.
- Na twarzy Jasia pojawia się mniej więcej taki wyraz.

![Jasio](https://paczaizm.pl/content/wp-content/uploads/matura-matematyka-poprawne-odpowiedzi-chory-dzieciak-dziecko-goraczka-lapie-sie-za-czolo.jpg)

- To co widzimy możemy prosto nazwać - stan.
- To co zarządza przejściem w stan A / B i czy może w niego przejść to maszyna stanów.

#### Maszyna stanów - na przykładzie apki

Przykład: https://pillarclient.z16.web.core.windows.net/app/templates/all

Z przykładu wyżej wiemy, że mamy w aplikacji kilka stanów:

- Idle - start,
- AppLoading - ładowanie apki,
- AppLoaded - apka załadowana,
- LoadingData - ładowanie danych,
- LoadedData - dane załadowano - widzimy je na interfejsie w postaci listy

Założmy, że jest jeszcze jeden stan, który odpowiada sytuacji błędu z serwera (LoadDataFail), wtedy chcemy pokazać jakiś ekran z błędem. Coś takiego:

![windows](https://i.ytimg.com/vi/PnD8fTtJw9I/maxresdefault.jpg)

#### Maszyna stanów - przykład w kodzie

```ts
const Idle: StateCreator<TemplatesViewStates> = () => ({ type: "IDLE" });
const AppLoading: StateCreator<TemplatesViewStates> = () => ({
  type: "APP_LOADING",
});
const AppLoaded: StateCreator<TemplatesViewStates> = () => ({
  type: "APP_LOADED",
});
const DataLoading: StateCreator<TemplatesViewStates> = () => ({
  type: "DATA_LOADING",
});
const DataLoaded: StateCreator<TemplatesViewStates> = <D>(data: D) => ({
  data,
  type: "DATA_LOADED",
});
const DataLoadFail: StateCreator<TemplatesViewStates> = () => ({
  type: "DATA_LOAD_FAIL",
});

const TEMPLATES_VIEW_ALLOWED_STATES: Partial<
  AllowedStatesMap<TemplatesViewStates>
> = {
  IDLE: [{ creator: AppLoading, type: "APP_LOADING" }],
  APP_LOADING: [{ creator: AppLoaded, type: "APP_LOADED" }],
  APP_LOADED: [{ creator: DataLoaded, type: "DATA_LOADED" }],
  // ...etc
};

const validateState = (
  states: StateRecord<string>[] | undefined,
  currentState: State<TemplatesViewStates, unknown>
) => {
  if (!Array.isArray(states) || states.length === 0) {
    throw new Error(
      `[LACK_OF_STATE_IN_ALLOWED_STATES] Check allowed state definition and align states for state with type ${currentState.type}`
    );
  }
};

const TemplatesViewStateMachine = (currentState = Idle()) => {
  return {
    next: <P extends Record<string, unknown>>(payload?: P) => {
      const allowedStates = TEMPLATES_VIEW_ALLOWED_STATES[currentState.type];

      validateState(allowedStates, currentState);

      const [state] = allowedStates;

      return TemplatesViewStateMachine(state.creator(payload));
    },
    set: <P extends Record<string, unknown>>(
      type: TemplatesViewStates,
      payload?: P
    ) => {
      const allowedStates = TEMPLATES_VIEW_ALLOWED_STATES[type];

      validateState(allowedStates, currentState);

      const state = allowedStates.find((s) => s.type === type);

      if (!state) {
        throw new Error(
          `[STATE_NOT_ALLOWED] Usage of ${type} state is not allowed for current state ${currentState.type}`
        );
      }

      return TemplatesViewStateMachine(state.creator());
    },
    read: () => currentState,
  };
};
// Idle -> AppLoading -> AppLoaded -> DataLoading -> DataLoaded
TemplatesViewStateMachine()
  .next()
  .next()
  .next()
  .next({ templates: [{ id: 1, name: "State machines" }] })
  .read();
 
TemplatesViewStateMachine.set(APP_LOADING) // przetworzy stan lub rzuci błąd
```

#### Maszyna stanów z xstate

```ts
const templatesViewMachine = createMachine({
  /* ... */
  context: {
    data: null,
  },
  states: {
    idle: {},
    selected: {
      initial: 'idle',
      states: {
        idle: {},
        appLoading: {},
        appLoaded: {},
        dataLoading: {},
        dataLoaded: {},
        dataLoadFail: {}
        ...etc
      }
    }
  },
  on: {
    /* ... */
  }
});

const [current, send] = useMachine(redditMachine);
current.matches('idle') // -> jaki jest aktualnie stan
const { subreddit, posts } = current.context;  // -> odczyt danych
```

#### Builder

Wzorzec wykorzystywany do budowania złożonych obiektów. Idealnie wpisuje się w budowanie stanu krok po kroku. Dodatkowo z wykorzystaniem wzorca **chain of responsibility** pozwala stworzyć wygodne i przejrzyste api. W poniższym przykładzie - tworzymy obiekt użytkownika, a następnie go modyfikujemy przed wykonaniem `valueOf()`.

```ts
interface User {
  username: string;
  phone: string;
  code: number;
}

const createUser = (): User => ({
  username: 'piotr1994',
  phone: '999 229 323',
  code: 2232,
});

const userBuilder = (user = createUser()) => ({
  valueOf: () => user, // albo build()
  setUsername: (username: User['username']) => userBuilder({ ...user, username }),
  setPhone: (phone: User['phone']) => userBuilder({ ...user, phone }),
  setCode: (code: User['code']) => userBuilder({ ...user, code }),
});

const basicUser = userBuilder();

// Component A
basicUser.setUsername('piotr').valueOf();

// Component B
basicUser.setUsername('Asia').valueOf()
```

#### Decorator

Tworzymy funkcje, która bierze obiekt serwisu oraz obiekt kontraktu i nadpisuje funkcje w serwisie dodając dodatkową implementację weryfikująca kontrakty na środowisku z odpowiednio ustawioną flagą. 

```ts
// Kontrakt i walidacja za pomocą biblioteki yup
const SearchServiceContract = {
    GET: {
        search: yup.object().shape({
            totalCount: yup.number().required(),
        })
    }
};

export default SearchServiceContract;
```

```ts
const checkContract = (promise, contract, region) => {
    return new Promise((resolve, reject) => {
        promise
            .then(response => {
                contract.validate(response, { strict: true }).catch(validationError => {
                    console.error(`Invalid data in ${region} contract`);
                    console.error(validationError);
                });
                resolve(response);
            })
            .catch(error => {
                reject(error);
            });
    });
};

// Implementacja wzorca dekorator
const applyContractsMiddleware = enabled => (contractsPromiseFn, serviceConfig) => {
    if (!enabled) {
        return serviceConfig;
    }

    const methodsKeys = Object.keys(serviceConfig);

    methodsKeys.forEach(methodKey => {
        const endpointsKeys = Object.keys(serviceConfig[methodKey]);

        endpointsKeys.forEach(endpointKey => {
            const endpoint = serviceConfig[methodKey][endpointKey];

            if (endpoint !== undefined) {
                serviceConfig[methodKey][endpointKey] = (...args) => {
                    const promise = endpoint(...args);

                    contractsPromiseFn().then(module => {
                        if (!module.default) {
                            throw new Error(
                                '[INVALID_MODULE_EXPORT] Contracts files must be default exported'
                            );
                        }

                        checkContract(
                            promise,
                            module.default[methodKey][endpointKey],
                            `${methodKey} -> ${endpointKey}`
                        );
                    });

                    return promise;
                };
            }
        });
    });

    return serviceConfig;
};
```

```ts
// Ustawienie kontraktów tylko na środowiskach z dedykowaną flagą.
export const applyNonProdContractsMiddleware = applyContractsMiddleware(__IS_CONTRACT_TESTING_ENABLED);
```

```ts
// Plik z serwisem i wykorzystanie middleware z walidacją kontraktu.
export const SearchService = createService(
    applyNonProdContractsMiddleware(() => import('./contracts/SearchServiceContract'), {
        GET: {
            search: params => get(ENDPOINTS.SEARCH, { params })
        }
    })
);
```

#### Chain of responsibility

Za każda kolejną operacją zwracamy nowo stworzony obiekt ze stanem po wykonanej operacji.

```ts
setUsername: (username: User['username']) => userBuilder({ ...user, username }) // w tej funkcji zawarty jest wzorzec
```

#### Observable

Prosty wzorzec, który polega na stworzeniu struktury danych, która przechowa nam funkcje. Je dodajemy za pomocą metody `subscribe`. Następnie przy wywołaniu operacji zmiany czegoś w obiekcie wywołujemy wszystkie funkcje ze struktury danych. 

> My nie piszemy tego ręcznie - użyjemy `rxjs`. To tylko przykład dla zobrazowania.

```ts
class UserModel {
    private _user: User;

    private _observers: UserObserver[] = [];

    get user(): User {
        return this._user;
    }

    set user(user: User) {
        this._user = user;
        this._notify();
    }

    private _notify = (): void => {
        this._observers.forEach(sb => sb(this.user));
    }

    subscribe = (observer: UserObserver): void => {
        this._observers = [...this._observers, observer];
    };

    unsubscribe = (observer: UserObserver): void => {
        this._observers = this._observers.filter(ob => ob !== observer);
    };

    load = (): Promise<void> => {
        return new Promise((resolve) => {
            setTimeout(() => {
                const user = { id: 1, firstName: 'Piotr', lastName: 'Piotrowicz' } as User;
                this.user = user;
                resolve();
            }, 500);
        });
    }
}

class UserView {
    constructor(private _model: UserModel) {
        this._model.subscribe(
            user => {
                this.render(user, document.getElementById('root'))
            }
        )
    }

    render = (user, target: HTMLElement): void => {
        const ul = document.createElement('ul');

        ul.appendChild(document.createRange().createContextualFragment(
            `
                <li>
                    <span>${user.id}</span>
                    <b>${user.firstName}</b>
                    <b>${user.lastName}</b>
                </li>`
        ));

        target.appendChild(ul);
    }
}

class UserController {
    private _view: UserView;
    private _model: UserModel;

    constructor() {
        this._model = new UserModel();
        this._view = new UserView(this._model);
    }

    display = async (): Promise<void> => {
        const isUserDisplayed = !!this._model.user;

        if (isUserDisplayed) {
            return;
        }

        await this._model.load();
    }
}

const userController = new UserController();
```

### Reaktywność, a state management.

Podejście reaktywne do zarządzania stanem daje bardzo dużo. Przykładowo załóżmy, że mamy zaimplementowaną maszyne stanów. Mamy mechanizm, który jest gdzieś w kodzie. Chciałbym na widoku A mieć obsługiwaną te samą maszyne stanów co w widoku B (te samą instancję). Dodatkowo widget C również ma z tego korzystać. Mam również middleware D, które chce dostać informację jeżeli pojawi się jakiś stan, który jest uważany z błędny i zaloguje wszystkie poprzednie stany użytkownika. 

Maszyna stanów świetnie sprawdza się ze wzorcem **observable**. Możemy zdefiniować cały proces przejścia, następnie go obserwować z różnych miejsc w aplikacji i odpowiednio pobierać informacje, które nas interesują, wykonywać na nich operacje. 

Analogicznie możemy wykorzystać podejście z **builderem**. 

Przykładowo `redux` implementuje wzorzec obserwator do tego, aby podłączyć się do **store** i nasłuchiwać na zmianę stanu.

```ts
store.subscribe(state => console.log(state))
```

Po wywoływaniu `dispatch` z obiektem akcji - `reducer` przejmie obiekt i zwróci nowy stan zaraz po tym wywołana zostanie funkcja anonimowa w `subscribe`. 

### Różne podejście do tego samego problemu, czyli nie zawsze pierwszy pomysł jest dobry.

https://stackblitz.com/edit/angular-gnxn4s?file=src%2Fapp%2Fonly-state-machine-used.component.ts

### Fabryki powtarzalnych funkcjonalności.

Mając na uwadze problemy, które zlokalizowaliśmy we wcześniejszych implementacjach możemy wykorzystać tą wiedze to zbudowania modułu obsługującego proste funkcjonalności `CRUD`.

Prototyp API:

```ts
type User = { id: number, name: string };
type Users = User[]

const MappingState = { type: 'mapping' }

const usersFeatures = Feature<Users>({ 
    create: service.addUsers,
    read: service.editUsers,
    update: service.updateUsers,
    delete: service.deleteUsers,
    value: [], // wartość domyślnie pusta tablica użytkowników
    states: [MappingState] // dodatkowe stany, które sobie definiujemy dla definicji typów,
    middlewares: [ErrorDrawMiddleware, ContractValidationMiddleware],
    events: [listenChange]
})

usersFeatures.setValue() // -> pozwala na zmiane wartości
usersFeatures.setValue((context) => context.value || context.state) // -> pozwala na zmiane wartości

usersFeatures.create() // -> wywołuje zapytanie do api z serwisu plus zmienia stany na **creating, created, createFail**. Analogicznie dla pozostałych.
usersFeatures.state // -> zwraca stan -> **idle**, **creating**, itp...
usersFeatures.value // -> zwraca wartość tablicę użytkowników.
usersFeatures.context // -> zwraca wartość oraz stan
usersFeatures.setState(MappingState) // -> ustawia stan manualnie
usersFeatures.onValueSet(() => {}) // -> funkcja wywoływana podczas modyfikowania value
```

### Funkcyjnie czy obiektowo

Wykorzystywanie obydwu paradygmatów powinno mieć swoje uzasadnienie. Przykładowo mając aplikację w `react` korzystać będziemy z paradygmatów funkcyjnych bo tak pisze się lepiej, szybciej, jest więcej gotowego kodu napisanego w ten sposób, ...etc. Jednak modelując aplikacje możemy dojść do wniosku, że obiektowe podejście gdzie nie gdzie jest wygodniejsze. Powinniśmy decydować o tym, które z nich wykorzystać do jakiego problemu, a nie dlatego, że jedne bardziej lubimy.

### Czy daleko nam do własnego frameworka?

Framework to nic innego jak zestaw podejść, dobrych praktyk, narzędzi, technologii. Większość z nich już mamy. Musimy tylko odpowiednio je dobierać do problemu. Wystarczy spisać gdzieś w jednym miejscu wszystkie reguły, które powinny być przestrzegane oraz stworzyć narzędzia - patrz `Feature`. 

Poniżej fragment dokumentu opisującego podejście do pisania kodu w projekcie oraz aspekty architektoniczne.

https://github.com/polubis/Billennium-workshops-03-State-management/blob/main/README_DS.md

#### Przykładowa implementacja w `react`

Poniższy kod jest kodem, który implementuje wyszukiwarkę podobną z działania do tej na `netflix`. Poniższy kod został również troche zmieniony. Nie mogę pokazać go w całości.

```ts
// Router - definicja pod jakimi metadanymi pokazujemy konkretne komponenty
const ROUTES = [
    AllContainer,
    HubsContainer,
    LiveContainer,
    NewsAndStoriesContainer,
    DocumentsContainer,
    SocialContainer
];

const SearchRouter = () => {
    const { activeRouteId } = useSearchRouterProvider();

    if (!ROUTES[activeRouteId]) {
        throw new Error('[COMPONENT_NOT_FOUND] There is no component defined for this route id.');
    }

    const Component = COMPONENTS[activeRouteId];

    return <Component />;
};
```

```ts
// RouterProvider definicja logiki routingu oraz udostępnienie jej przez context api
export const ROUTES = ['All', 'Hubs', 'Live', 'News and stories', 'Documents', 'Social'];
export const [
    ALL_ROUTE,
    HUBS_ROUTE,
    LIVE_ROUTE,
    NEWS_AND_STORIES_ROUTE,
    DOCUMENTS_ROUTE,
    SOCIAL_ROUTE
] = ROUTES;

const SearchRouterContext = createContext();

const isRoute = (id, routes, route) => id === routes.indexOf(route);

export const SearchRouterProvider = ({ children }) => {
    const [activeRouteId, setActiveRouteId] = useState(0);

    const changeRoute = useCallback(value => {
        if (typeof value === 'number') {
            if (!ROUTES[value]) {
                throw new Error('[INVALID_ROUTE_ID] Route id is not defined in routes.');
            }

            setActiveRouteId(value);
        }

        const idx = ROUTES.findIndex(route => route === value);

        if (idx === -1) {
            throw new Error('[INVALID_ROUTE] Route name is not defined in routes.');
        }

        setActiveRouteId(idx);
    }, []);

    const value = useMemo(
        () => ({
            activeRoute: ROUTES[activeRouteId],
            activeRouteId,
            routes: ROUTES,
            isAllRoute: () => isRoute(activeRouteId, ROUTES, ALL_ROUTE),
            changeRoute
        }),
        [activeRouteId]
    );

    return <SearchRouterContext.Provider value={value}>{children}</SearchRouterContext.Provider>;
};

export const useSearchRouterProvider = () => useContext(SearchRouterContext);
```

```ts
// Moduł - punkt wejścia do konkretnej większej funkcjonalności
const SearchModuleContainer = WithSuspense(() =>
    import('./containers/SearchModuleContainer').then(module => ({
        default: module.SearchModuleContainer
    }))
);

const SearchModule = () => {
    const [isOpen] = useSearchModule();

    return isOpen ? <SearchModuleContainer /> : null;
};
```

```ts
// Prosty hook do obsługi pokaż moduł / schowaj moduł
const opened = new BehaviorSubject(false);
export const opened$ = opened.asObservable();

export const openSearchModule = () => {
    opened.next(true);
};

export const closeSearchModule = () => {
    opened.next(false);
};

export const useSearchModule = () => {
    const [isOpen, setIsOpen] = useState(opened.getValue());

    useEffect(() => {
        const sub = opened$.subscribe(open => setIsOpen(open));

        return () => {
            sub.unsubscribe();
        };
    }, []);

    return [isOpen, openSearchModule, closeSearchModule];
};
```

```ts
// Encja - obsługa logiki biznesowej oraz stanów przejściowych
export const SEARCH_DATA_KEYS = {
    BLOGS: 'blogs',
    DISCUSSIONS: 'discussions',
    DOCUMENTS: 'documents',
    EVENT_FAQ_ITEMS: 'eventFaqItems',
    EVENT_MEMBERS: 'eventMembers',
    EVENTS: 'events',
    LINKS: 'links',
    LIVE_SESSIONS: 'liveSessions',
    MESSAGES: 'messages',
    TEXT_BOXES: 'textBoxes',
    VIDEOS: 'videos'
};

const createStateForNodes = (data, stateFn, keys = SEARCH_DATA_KEYS) =>
    Object.values(keys).reduce((acc, key) => merge(acc, { [key]: stateFn(key, data[key]) }), {});

const checkIsAllLoaded = ({ results, totalCount }) => results.length >= totalCount;

const createEmptyNodes = () =>
    Object.values(SEARCH_DATA_KEYS).reduce(
        (acc, key) =>
            merge(acc, {
                [key]: {
                    totalCount: 0,
                    results: []
                }
            }),
        {}
    );

const EMPTY_NODES = createEmptyNodes();

const preventEmptyNodes = data => merge(EMPTY_NODES, data);

export class SearchDataEntity extends Entity {
    _statelessValue;

    constructor() {
        super(Idle());
    }

    asLoading = () => this._replace(Loading());

    asLoadFail = () => this._replace(LoadFail());

    _setTransitionState = (keys, stateFn) =>
        this._merge({
            data: merge(
                this.valueOf().data,
                keys.reduce(
                    (acc, key) => merge(acc, { [key]: stateFn(this._statelessValue[key]) }),
                    {}
                )
            )
        });

    asLoaded = data => {
        this._statelessValue = preventEmptyNodes(data);

        return this._replace(
            Loaded(
                merge(
                    { totalCount: this._statelessValue.totalCount },
                    createStateForNodes(this._statelessValue, (_, nodeData) =>
                        checkIsAllLoaded(nodeData) ? LoadedAll(nodeData) : Loaded(nodeData)
                    )
                )
            )
        );
    };

    asLoadingMore = keys => this._setTransitionState(keys, LoadingMore);

    asLoadedMore = (keys, data) => {
        const notEmptyData = preventEmptyNodes(data);

        this._statelessValue = merge(
            this._statelessValue,
            keys.reduce(
                (acc, key) =>
                    merge(acc, {
                        [key]: merge(this._statelessValue[key], {
                            results: [
                                ...this._statelessValue[key].results,
                                ...notEmptyData[key].results
                            ]
                        })
                    }),
                {}
            )
        );

        return this._merge({
            data: merge(
                this.valueOf().data,
                createStateForNodes(this._statelessValue, (_, nodeData) =>
                    checkIsAllLoaded(nodeData) ? LoadedAll(nodeData) : LoadedMore(nodeData)
                )
            )
        });
    };

    asLoadMoreFail = keys => this._setTransitionState(keys, LoadMoreFail);

    asSorting = keys => this._setTransitionState(keys, Sorting);

    asSorted = (keys, data) => {
        this._statelessValue = merge(
            this._statelessValue,
            keys.reduce(
                (acc, key) =>
                    merge(acc, {
                        [key]: data[key]
                    }),
                {}
            )
        );

        return this._merge({
            data: merge(
                this.valueOf().data,
                createStateForNodes(this._statelessValue, (_, nodeData) =>
                    checkIsAllLoaded(nodeData) ? LoadedAll(nodeData) : Sorted(nodeData)
                )
            )
        });
    };

    asSortingFail = keys => this._setTransitionState(keys, SortingFail);

    isBusy = keys => {
        const { data, isBusy } = this.valueOf();
        return isBusy || keys.some(key => data[key].isBusy);
    };

    isLoadedAll = keys => {
        const { data } = this.valueOf();
        return keys.some(key => data[key].isLoadedAll);
    };

    getLength = key => this._statelessValue[key].results.length;

    statelessValueOf = () => this._statelessValue;
}
```

```ts 
// Provider - zarządzanie komunikacją, abstrakcja pomiędzy encjami a komponentami oraz udostępnienie handlerów przez context api do wywołania całej sekwencji operacji. Również zarządzanie asynchronicznościa.
const SearchContext = createContext();

export const SearchProvider = ({ children }) => {
    const [payloadEntity, setPayloadEntity] = useState(new SearchPayloadEntity());
    const [searchDataEntity, setSearchDataEntity] = useState(new SearchDataEntity());

    const [search, search$] = useSubject();
    const [searchingMore, searchingMore$] = useSubject();
    const [sorting, sorting$] = useSubject();

    useEffect(() => {
        const subs = new Subscription();

        subs.add(
            search$
                .pipe(
                    map(payloads =>
                        merge(payloads, {
                            payloadEntity: payloads.payloadEntity
                                .setDefaults()
                                .setQuery(payloads.query)
                        })
                    ),
                    filter(({ payloadEntity }) => payloadEntity.isValid()),
                    debounceTime(300),
                    map(payloads =>
                        merge(payloads, {
                            searchDataEntity: payloads.searchDataEntity.asLoading()
                        })
                    ),
                    tap(({ payloadEntity, searchDataEntity }) => {
                        setPayloadEntity(payloadEntity);
                        setSearchDataEntity(searchDataEntity);
                    }),
                    switchMap(({ searchDataEntity, payloadEntity }) =>
                        from(
                            SearchService.GET.search(
                                merge(SearchPayloadEntity.getDefaults(), {
                                    query: payloadEntity.valueOf().query
                                })
                            )
                        ).pipe(
                            tap(response => {
                                setSearchDataEntity(searchDataEntity.asLoaded(response));
                            }),
                            catchError(() => {
                                setSearchDataEntity(searchDataEntity.asLoadFail());
                                return EMPTY;
                            })
                        )
                    )
                )
                .subscribe()
        );

        subs.add(
            searchingMore$
                .pipe(
                    filter(
                        payloads =>
                            !payloads.searchDataEntity.isBusy(payloads.searchParams.dataKeys) &&
                            !payloads.searchDataEntity.isLoadedAll(payloads.searchParams.dataKeys)
                    ),
                    map(payloads =>
                        merge(payloads, {
                            searchDataEntity: payloads.searchDataEntity.asLoadingMore(
                                payloads.searchParams.dataKeys
                            ),
                            payloadEntity: payloads.payloadEntity.incrementOffsets(
                                payloads.searchParams.dataKeys
                            )
                        })
                    ),
                    tap(payloads => {
                        setSearchDataEntity(payloads.searchDataEntity);
                        setPayloadEntity(payloads.payloadEntity);
                    }),
                    switchMap(({ searchDataEntity, payloadEntity, searchParams }) =>
                        from(
                            SearchService.GET.search(
                                merge(SearchPayloadEntity.getDefaults(), {
                                    query: payloadEntity.valueOf().query,
                                    offset: payloadEntity.getOffset(searchParams.dataKeys[0]),
                                    types: searchParams.searchTypes.join(','),
                                    sortBy: payloadEntity.getSortBy(searchParams.dataKeys[0])
                                })
                            )
                        ).pipe(
                            tap(response => {
                                setSearchDataEntity(
                                    searchDataEntity.asLoadedMore(searchParams.dataKeys, response)
                                );
                            }),
                            catchError(() => {
                                setSearchDataEntity(
                                    searchDataEntity.asLoadMoreFail(searchParams.dataKeys)
                                );
                                return EMPTY;
                            }),
                            takeUntil(search$)
                        )
                    )
                )
                .subscribe()
        );

        subs.add(
            sorting$
                .pipe(
                    filter(
                        payloads =>
                            !payloads.searchDataEntity.isBusy(payloads.searchParams.dataKeys)
                    ),
                    map(payloads =>
                        merge(payloads, {
                            payloadEntity: payloads.payloadEntity.setSortBy(
                                payloads.searchParams.dataKeys,
                                payloads.sortBy
                            ),
                            searchDataEntity: payloads.searchDataEntity.asSorting(
                                payloads.searchParams.dataKeys
                            )
                        })
                    ),
                    tap(payloads => {
                        setSearchDataEntity(payloads.searchDataEntity);
                        setPayloadEntity(payloads.payloadEntity);
                    }),
                    switchMap(({ searchDataEntity, payloadEntity, searchParams }) =>
                        from(
                            SearchService.GET.search(
                                merge(SearchPayloadEntity.getDefaults(), {
                                    query: payloadEntity.valueOf().query,
                                    types: searchParams.searchTypes.join(','),
                                    sortBy: payloadEntity.getSortBy(searchParams.dataKeys[0]),
                                    limit: searchDataEntity.getLength(searchParams.dataKeys[0])
                                })
                            )
                        ).pipe(
                            tap(response => {
                                setSearchDataEntity(
                                    searchDataEntity.asSorted(searchParams.dataKeys, response)
                                );
                            }),
                            catchError(() => {
                                setSearchDataEntity(
                                    searchDataEntity.asSortingFail(searchParams.dataKeys)
                                );
                                return EMPTY;
                            }),
                            takeUntil(search$)
                        )
                    )
                )
                .subscribe()
        );

        return () => {
            subs.unsubscribe();
        };
    }, []);

    const startSearching = query => {
        search.next({ payloadEntity, searchDataEntity, query });
    };

    const startSearchingMore = searchParams => {
        searchingMore.next({
            payloadEntity,
            searchDataEntity,
            searchParams
        });
    };

    const startSorting = (searchParams, sortBy) => {
        sorting.next({
            payloadEntity,
            searchDataEntity,
            searchParams,
            sortBy
        });
    };

    const value = useMemo(() => {
        return {
            searchData: searchDataEntity.valueOf(),
            statelessSearchData: searchDataEntity.statelessValueOf(),
            payload: payloadEntity.valueOf(),
            startSearching,
            startSearchingMore,
            startSorting
        };
    }, [payloadEntity, searchDataEntity]);

    return <SearchContext.Provider value={value}>{children}</SearchContext.Provider>;
};

export const useSearchProvider = () => useContext(SearchContext);
```

```ts
// Serwis - logika komunikacji z serwerem, local storage, ciastkami, session storage, indexed db, ...itp
const { get, createService } = Api;

export const SearchService = createService(
    applyNonProdContractsMiddleware(() => import('./contracts/SearchServiceContract'), {
        GET: {
            search: params => get(ENDPOINTS.SEARCH, { params })
        }
    })
);

```

```ts 
ServiceContract - kontrakt, który weryfikuje czy aplikacja korzysta z aktualnych modeli.
const createNodeContract = contract =>
    yup
        .object()
        .shape({
            totalCount: yup.number().required(),
            results: yup
                .array()
                .of(
                    yup
                        .object()
                        .shape(contract)
                        .required()
                )
                .required()
        })
        .notRequired();

const SearchServiceContract = {
    GET: {
        search: yup.object().shape({
            totalCount: yup.number().required(),
            blogs: createNodeContract({
                content: yup.string().required(),
                createdAt: yup.string().required(),
                eventName: yup.string().required(),
                id: yup.number().required(),
                isPublic: yup.bool().required(),
                readingTime: yup.number().required(),
                thumbnail: yup.string().nullable(),
                title: yup.string().required(),
                url: yup.string().required()
            }),
        })
    }
};

export default SearchServiceContract;
```

### Podsumowanie

Przeszliśmy przez kilka implementacji i rozwiązań tego samego problemu. Każde z nich ma swoje dobre i słabe strony. Kwestią doświadczenia jest dobór odpowiedniego rozwiązania dlatego nigdy nie powinniśmy zamykać się na własne eksperymenty. 

### FAQ

- Q: Czy redux to maszyna stanów skoro obiekty akcji posiadają typ?
- A: Nie. Traktujmy `reduxa` jak `Context API` w `react` czy `service` w `angular`. To tylko narzędzie do ustalenia co ma się stać w oparciu o jaki obiekt akcji. Aby to była maszyna stanów brakuje nam obsługi kiedy w jaki stan możemy przejść. Jest to jak najbardziej możliwe do implementacji.

- Q: Czy zaprezentowane podejścia będą się sprawdzały w każdej aplikacji?
- A: Nie. Przykładowo zastosowanie podejścia z `Manager` świetnie sprawdzi się w aplikacjach posiadających wiele podobnych funkcjonalności. Jednak w momencie gdy dochodzi dużo customowej logiki może okazać się, że łatwiej napisać to bez `Manager` np. z wykorzystaniem `redux`. 

- Q: Dlaczego mam tworzyć koło na nowo skoro są już dostępne narzędzia?
- A: Odpowiedź na to jest prosta. Dla nauki, zrozumienia narzędzi, których się używa i jakie problemy rozwiązują oraz po to, żeby znajdować zawsze najlepsze rozwiązanie. Później mając przed sobą projekt odrazu można oszacować, która technologia najlepiej się sprawdzi i dlaczego.

