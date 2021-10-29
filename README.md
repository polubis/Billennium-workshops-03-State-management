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
