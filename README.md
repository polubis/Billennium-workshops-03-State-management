# Billennium-workshops-03-State-management-and-application-flow/architecture

Warsztaty wprowadzające w różne techniki zarządzania stanem i architekture aplikacji frontendowej.

### Czym jest state - na przykładzie Jasia ?

- Dla uproszczenia przyjmijmy, że rozmawiamy na temat matury z Jasiem.
- Tłumaczymy mu jakieś skomplikowane zagadnienie.
- Na twarzy Jasia pojawia się mniej więcej taki wyraz.

![Jasio](https://paczaizm.pl/content/wp-content/uploads/matura-matematyka-poprawne-odpowiedzi-chory-dzieciak-dziecko-goraczka-lapie-sie-za-czolo.jpg)

- To co widzimy możemy prosto nazwać - stan.
- To jakie procesy zadziały się pomiędzy stanem A, a stanem B są dla nas nieistotne.

### Stan w aplikacji

Przykład: https://pillarclient.z16.web.core.windows.net/app/templates/all

- Mamy aplikację z listą templatek.
- Początkowo aplikacja jest w stanie uruchamiania - widzimy napis "Please wait a second".
- Po pobraniu kodu apki - widzimy animacje ładowania - stan ładowania danych.
- Po załadowaniu jest kolejny stan - stan wyświetlania danych.

To co się dzieje wewnątrz, jakie mechanizmy tym sterują nas nie interesuje. Ważne jest tylko to co widzimy na interfejsie - tak jak na twarzy Jasia.

### State management

Proces zarządzania zmianą stanów. Tu skupimamy się na konkretach. Przykładowo móżg plus hormony sterują zachowaniem organizmu u człowieka. W przypadku aplikacji może to być `react` + wzorzec maszyny stanów zaimplementowany samodzielne bądź z użyciem gotowej biblioteki jak `xstate`.

### Wzorce projektowe na ratunek

Tak jak do biegania potrzebne są dobre buty to i w przypadku zarządzania stanem potrzebujemy odpowiednich narzędzi. Będą nimi wzorce projektowe. To czy korzystamy z React, Angular, ...etc nie ma żadnego znaczenia,

#### Maszyna stanów na przykładzie jas.jpg

Co steruje procesem zmiany wyrazu twarzy Jasia? No z pewnością jest to mózg (pewnie nie jest to tak proste, ale wiadomo). 

Maszyna stanów to nic innego jak właśnie mózg, który steruje procesem przejścia oraz wnioskuje, w które stany aktualnie można przejść, a w które nie. Przykładowo żeby zacząć biec najpierw trzeba iść.

#### Maszyna stanów na przykładzie apki

Przykład: https://pillarclient.z16.web.core.windows.net/app/templates/all

Z przykładu wyżej wiemy, że mamy w aplikacji kilka stanów:

- Idle - start,
- AppLoading - ładowanie apki,
- AppLoaded - apka załadowana,
- LoadingData - ładowanie danych,
- LoadedData - dane załadowano - widzimy je na interfejsie w postaci listy

Założmy, że jest jeszcze jeden stan, który odpowiada sytuacji błędu z serwera (LoadDataFail), wtedy chcemy pokazać jakiś ekran z błędem. Coś takiego:

![windows](https://i.ytimg.com/vi/PnD8fTtJw9I/maxresdefault.jpg)

Z logicznego punktu widzenia żeby przejść w stan błędu pobierania danych najpierw trzeba przejść przez stany Idle, AppLoading, AppLoaded, LoadingData, tu będzie stan LoadDataFail.

Tym w jakie stany można przejść steruje właśnie maszyna stanów.

#### Maszyna stanów - przykład w kodzie

1. Zanim zaczniesz kodować - zidentyfikuj stany w twojej funkcjonalności.
2. Określ przejścia pomiędzy stanami. Czy stan A może przejść w stan B.
3. Zacznij implementacje krótkich funkcji tworzących poszczególne stany.
4. Dorzuć logikę obsługującą zmiane - jak przejście ze stanu A w C jest nie dozwolone, to rzuć wyjątek (podejście **fail fast**).

> Pamiętaj, że narazie to tylko POC naszej maszyny stanów - pierwszy pomysł, a nie gotowa implementacja. Zamiast pisać samemu możemy użyc gotowego rozwiązania w zależności od potrzeb.

```ts
interface State<T extends string, D> {
  data?: D;
  type: T;
}

type StateCreator<T extends string> = <D>(data?: D) => State<T, D>;

type StateRecord<T extends string> = { creator: StateCreator<T>; type: T };

type AllowedStatesMap<T extends string> = {
  [K in T]: StateRecord<T>[];
};

type TemplatesViewStates =
  | "IDLE"
  | "APP_LOADING"
  | "APP_LOADED"
  | "DATA_LOADING"
  | "DATA_LOADED"
  | "DATA_LOAD_FAIL";

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
    restart: () => TemplatesViewStateMachine(Idle()),
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


```ts
Tu dodać przykłąd z veh
```

#### Chain of responsibility

Za każda kolejną operacją zwracamy nowo stworzony obiekt ze stanem po wykonanej operacji.

```ts
setUsername: (username: User['username']) => userBuilder({ ...user, username }) // w tej funkcji zawarty jest wzorzec
```

#### Observable

#### Inversion of control


### Przepis na "dobry" state management

Niestety takiego nie ma. Żadne z podejść nie jest idealne - powinno być dobierane per projekt i zespół developerski. To prześledzimy na koniec.

### Reaktywność, a state management.

Podejście reaktywne do zarządzania stanem daje bardzo dużo. Przykładowo załóżmy, że mamy zaimplementowaną maszyne stanów. Mamy mechanizm, który jest gdzieś w kodzie. Chciałbym na widoku A mieć obsługiwaną te samą maszyne stanów co w widoku B (te samą instancję). Dodatkowo widget C również ma z tego korzystać. Mam również middleware D, które chce dostać informację jeżeli pojawi się jakiś stan, który jest uważany z błędny i zaloguje wszystkie poprzednie stany użytkownika. 

Maszyna stanów świetnie sprawdza się ze wzorcem **observable**. Możemy zdefiniować cały proces przejścia, następnie go obserwować z różnych miejsc w aplikacji i odpowiednio pobierać informacje, które nas interesują, wykonywać na nich operacje. 

Analogicznie możemy wykorzystać podejście z **builderem**. 

Przykładowo `redux` implementuje wzorzec obserwator do tego, aby podłączyć się do **store** i nasłuchiwać na zmianę stanu.

```ts
store.subscribe(state => console.log(state))
```

Po wywoływaniu `dispatch` z obiektem akcji - `reducer` przejmie obiekt i zwróci nowy stan zaraz po tym wywołana zostanie funkcja anonimowa w `subscribe`. 

![Redux](https://res.cloudinary.com/practicaldev/image/fetch/s--V1XmAEPc--/c_imagga_scale,f_auto,fl_progressive,h_900,q_auto,w_1600/https://i.stack.imgur.com/LNQwH.png)

### Różne podejście do tego samego problemu, czyli nie zawsze pierwszy pomysł jest dobry.



### Analiza przykładu prostej apki i różnych rozwiązań tego samego problemu

https://stackblitz.com/edit/angular-gnxn4s?file=src%2Fapp%2Fonly-state-machine-used.component.ts

### Fabryki powtarzalnych funkcjonalności.

### Funkcyjnie czy obiektowo

### Czy daleko nam do własnego frameworka?

### FAQ

Q: Czy redux to maszyna stanów skoro obiekty akcji posiadają typ ?
A: Nie. Traktujmy `reduxa` jak `Context API` w `react` czy `service` w `angular`. To tylko narzędzie do ustalenia co ma się stać w oparciu o jaki obiekt akcji. Aby to była maszyna stanów brakuje nam obsługi kiedy w jaki stan możemy przejść. Jest to jak najbardziej możliwe do implementacji.

### Podsumowanie
