# Billennium-workshops-03-State-management-and-application-flow/architecture

Warsztaty wprowadzające w różne techniki zarządzania stanem. 

### Jak uruchomić przykład ?

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

#### Builder

Wzorzec wykorzystywany do budowania złożonych obiektów. Idealnie wpisuje się w budowanie stanu krok po kroku. Dodatkowo z wykorzystaniem wzorca **chain of responsibility** pozwala stworzyć wygodne i przejrzyste api.

```ts

```

#### Decorator


```ts

```

#### Chain of responsibility

```ts

```

#### Inversion of control


### Przepis na "dobry" state management

Niestety takiego nie ma. Żadne z podejść nie jest idealne - powinno być dobierane per projekt i zespół developerski. To prześledzimy na koniec.

### Reaktywność, a state management.

### Fabryki powtarzalnych funkcjonalności.

### Funkcyjnie czy obiektowo

### Różne podejście do tego samego problemu, czyli nie zawsze pierwszy pomysł jest dobry.

### Analiza przykładu prostej apki i różnych rozwiązań tego samego problemu

### Czy daleko nam do własnego frameworka?

### Podsumowanie
