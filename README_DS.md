# General naming conventions & good practices

This document will be improved from time to time.

## Avoid default exports if possible

Default exports increases boilerplate connected to re-exporting and also makes all refactors bigger.
Also developer can make typo with default name of imported being.

```js
// Don't do
export default Component;
// Do
export const Component = () => {};
export { Component };
```

## No name shadowing

This can produce really hard to track bugs. Always add somekind of prefix in function which describes
new name or move this function to other context without duplicated names.

```js
// Don't do
const fn = a => {
    const handleA = a => {};
};
// Do
const fn = a => {
    const handleA = newA => {};
};
```

## Don't use destructuring in function React components

Usage of destructuring creates line breaks and bigger git differences. Especially in
functional component types. Also in big components this creates confiusing in names
recognition process.

```js
// Don't do
const Component = ({ a, b, c, d, e, f, g, i, j, k }) => {};
// Do
const Component = props => {
    const { a } = props;
    // Use when needed.
};
```

## Create index.js files

This reduces imports boilerplate and makes refactors easier.

`index.js` file should be created always for dedicated folder.

## Encapsulate everything what is not connected to presentation

For example instead of coding something inside `react` components create separate being which allows to perform some logic and return this value. After that save state in `react`. This allows to save time with later refactors, logic change.
Logic will be changed inside single function not in all application modules.

```js
// Don't do
const Event = () => {
    return (
        <>
            {event.type === EVENT_TYPE.TYPE && <div></div>}
            {events
                .filter(event => event.type !== EVENT_TYPE)
                .map(component => (
                    <li></li>
                ))}
        </>
    );
};

// Do
const Event = () => {
    return (
        <>
            {EventEntity.isConcretteType() && <div></div>}
            {EventsEntity.getConcretteTyped().map(component => (
                <li></li>
            ))}
        </>
    );
};
```

## Handle keyword usage

Every event handler must be named with `handle` prefix. `Handle` keyword is right now reserved only for **event handlers**.
This allows to easy determine which function is used for dedicated handlers. Also additional logic can be added easier.
**event handler** just manages whole process. Logic is coded in other functions.

```js
// Don't do
const openDialog = () => {
    // ...
};
return <Button onClick={openDialog} />;
// Do
const handleButtonClick = () => {
    // ...
};
return <Button onClick={handleButtonClick} />;
```

# Architecture conventions

## Elements naming

All architecture elements must be named with `postfix` which describes type of element in architecture.
Our architecture elements and examples:

-   Modules -> EventModule,
-   Routers -> EventRouter,
-   Providers -> UserDataProvider,
-   RouterProvider - UserRouterProvider
-   Containers -> EventContainer,
-   Components -> PanelComponent,
-   Entities -> SearchDataEntity
-   Contracts -> SearchServiceContract

## Elements structure

Our structure should will be flat. It means:

```js
// search
components
   -> index.js
   -> layout
       -> index.js
       -> LayoutComponent.js
       -> LayoutComponent.module.scss - optional
       -> LayoutComponent.stories.js - optional
routing
   -> index.js
   -> SearchRouter.js
services
   -> index.js
   -> SearchService
   -> contracts
      SearchServiceContract.js
providers
   -> index.js
   -> SearchProvider
entities
   -> SearchDataEntity.js
   -> index.js
containers
   -> index.js
   -> SearchContainer.js
```

## Modules

`Module` means starting point of feature.

**Must contain**:

-   Statements which shows or hides module (authorization guard, or other if statement)

**Can contain**:

-   Layout components.
-   Data providers.
-   Some additional logic to display content basing of open/close flags.
-   Routers

**Cannot contain**:

-   Direct implementations of something. fe:

```js
// Don't do
const [open, setOpen] = useState(false);

const handleOpen = () => {
    setOpen(true);
};
// Do
const [open, toggleOpen] = useOpenFeature();
// With that in tests we can full mock useOpenFeature and write easy integration tests.
```

-   Direct style definitions. Instead of that we create always `Layout` components or something similar and we use this component inside `Module`.

## Routers

`Router` **only** defines routing logic - in simple words components / route map.

**Must contain**:

-   Routes definition via `react-router` or via variables.
-   Usage of router provider hook.

**Can contain**:

-   Additional guard definitions based on authorization or something similar.
-   Routing validation logic.

**Cannot contain**:

-   Direct implementations of other things like data management, style definitions.

## RouterProvider

`RouterProvider` **only** defines routing logic and exposes this logic via context.

**Must contain**:

-   Routing logic.
-   Context to expose api and current routing state.

**Can contain**:

-   Additional guard definitions based on authorization or something similar.
-   Routing validation logic.

**Cannot contain**:

-   Direct implementations of other things like data management, style definitions.

## Containers

Name of component which consumes business logic via `Providers` or `Directly` and passes all needed data, event handlers to child presentational beings which we call in our architecture `Components`.

Containers just consumes `Providers` or `Entities` and connects their API. After that expose facade api for `Components`.

**Must contain**:

-   Connection with business logic via `Providers` or `Entities`.
-   `children` prop.

**Can contain**:

-   Additional specific authorization logic or business logic. Like showing button for `admins` and others...

**Cannot contain**:

-   Direct implementations of other things like data management, style definitions.

## Contracts

`Contracts` define shape of response model from backend endpoint.

**Must contain**:

-   Default exported contract shape.
-   One file per contract

**Can contain**:

-   Helper functions or partial contract shapes for readability.

**Cannot contain**:

-   Cannot be re-exported.

## Providers

This being role is to consume `Entities` and create API which allows to trigger dedicated change and propagate change to all listening components via `Context API`.

Provider is only mediator between `Container` and `Entity`.

**Must contain**:

-   Business logic consumption via `Entites` or other value objects.
-   Data propagation mechanism via `Context API` or `rxjs subjects`.

**Cannot contain**:

-   Direct implementations of other things like data management, style definitions.

## Entites

Represents models with their domain / business logic. `Entities` uses `value objects` from `DDD`. By convention every `entity` contain `valueOf()` method which returns current value and disables option to trigger next changes.

`valueOf()` should be used by `Providers` to hide implementation details of entities and allow `Containers` to read pure data.

**Must contain**:

-   `valueOf()` not static, `from()` optional static, methods.
-   all model modification logic, validation logic.

**Can contain**:

-   Dependencies to other `Entities` or helper functions.

**Cannot contain**:

-   Async operations and side effects.

## Services

Beings which allows to determine logic of load data from servers or other type of data sources like `Local Storage`, `Cookies`, `Service workers`.

**Must contain**:

-   logic of retreving data from dedicated source.
-   async operations or side effects.

**Can contain**:

-   Dependencies to other `Services`.
-   Error / response formatting logic.
-   Observable based API to determine current errors state.
