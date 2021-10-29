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

### Maszyna stanów na przykładzie jas.jpg

Co steruje procesem zmiany wyrazu twarzy Jasia? No z pewnością jest to mózg (pewnie nie jest to tak proste, ale wiadomo). 

Maszyna stanów to nic innego jak właśnie mózg, który steruje procesem przejścia oraz wnioskuje, w które stany aktualnie można przejść, a w które nie. Przykładowo żeby zacząć biec najpierw trzeba iść.

### Maszyna stanów na przykładzie apki

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

#### Przykład w kodzie

> Pamiętaj, że narazie to tylko POC naszej maszyny stanów - pierwszy pomysł, a nie gotowa implementacja.

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
