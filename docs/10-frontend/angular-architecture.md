# Angular Frontend Architecture

## Overview

ThingsBoard's frontend is a sophisticated single-page application built on **Angular 18.2.13** with Material Design components. The architecture follows modular design principles with lazy-loaded feature modules, centralized state management via **NgRx**, real-time WebSocket communication, and an extensible widget framework. This design enables a responsive, scalable user interface that handles complex IoT dashboard scenarios.

## Technology Stack

| Technology | Version | Purpose |
|------------|---------|---------|
| Angular | 18.2.13 | Core framework |
| Angular Material | 18.2.10 | UI component library |
| NgRx | 18.x | State management |
| RxJS | 7.8.x | Reactive programming |
| TypeScript | 5.5.x | Type safety |
| angular-gridster2 | 18.x | Dashboard grid layout |
| Chart.js | 4.4.x | Charting library |
| Leaflet | 1.9.x | Map widgets |

## Key Behaviors

1. **Modular Structure**: Core, shared, and feature modules with clear separation of concerns.

2. **Centralized State**: NgRx store manages application state with actions, reducers, and effects.

3. **Real-Time Data**: WebSocket services deliver live telemetry and notifications to widgets.

4. **Dynamic Widgets**: Widget components are dynamically loaded and receive data via subscriptions.

5. **Lazy Loading**: Feature modules load on-demand to reduce initial bundle size.

6. **Role-Based Access**: Routes and features are protected based on user authority.

## Module Architecture

### Module Hierarchy

```mermaid
graph TB
    subgraph "Root"
        APP[AppModule]
    end

    subgraph "Core Module (Singleton)"
        CORE[CoreModule]
        AUTH[Auth Services]
        HTTP[HTTP Services]
        WS[WebSocket Services]
        STORE[NgRx Store]
    end

    subgraph "Shared Module"
        SHARED[SharedModule]
        COMP[Components]
        DIR[Directives]
        PIPE[Pipes]
        MODEL[Models]
    end

    subgraph "Feature Modules (Lazy)"
        LOGIN[LoginModule]
        HOME[HomeModule]
        DASH[DashboardModule]
    end

    APP --> CORE
    APP --> SHARED
    APP --> LOGIN
    APP --> HOME
    HOME --> DASH
```

### Module Responsibilities

| Module | Purpose | Loading |
|--------|---------|---------|
| AppModule | Bootstrap, root routing | Immediate |
| CoreModule | Singleton services, state, interceptors | Immediate |
| SharedModule | Reusable components, pipes, directives | Immediate |
| LoginModule | Authentication flows | Lazy |
| HomeModule | Main authenticated area | Lazy |
| DashboardModule | Dashboard features | Lazy |

### Core Module Components

```mermaid
graph TB
    subgraph "CoreModule"
        subgraph "State Management"
            STORE[NgRx Store]
            ACTIONS[Actions]
            REDUCERS[Reducers]
            EFFECTS[Effects]
            SELECTORS[Selectors]
        end

        subgraph "Services"
            AUTH_SVC[AuthService]
            HTTP_SVC[HTTP Services]
            WS_SVC[WebSocket Services]
            UTIL_SVC[Utility Services]
        end

        subgraph "Security"
            GUARD[Route Guards]
            INTERCEPT[Interceptors]
        end
    end

    STORE --> ACTIONS
    ACTIONS --> REDUCERS
    ACTIONS --> EFFECTS
    REDUCERS --> SELECTORS
```

### Shared Module Components

| Category | Examples | Purpose |
|----------|----------|---------|
| Components | Entity selectors, dialogs, buttons | Reusable UI elements |
| Directives | Truncate, context menu, ellipsis | DOM manipulation |
| Pipes | File size, time format, enum display | Data transformation |
| Models | Entity types, widget models | Type definitions |
| Services | Image, resource, notification | Shared business logic |

## Component Architecture

### Component Hierarchy

```mermaid
graph TB
    subgraph "Application Shell"
        APP[AppComponent]
    end

    subgraph "Authenticated Area"
        HOME[HomeComponent]
        MENU[SideMenuComponent]
        NOTIFY[NotificationBell]
    end

    subgraph "Feature Pages"
        DEVICES[DevicesPage]
        ASSETS[AssetsPage]
        RULES[RuleChainsPage]
        DASHBOARDS[DashboardsPage]
    end

    subgraph "Dashboard Runtime"
        DASH[DashboardComponent]
        GRID[GridsterContainer]
        WIDGET_WRAP[WidgetContainer]
        WIDGET[WidgetComponent]
        DYN_WIDGET[DynamicWidget]
    end

    APP --> HOME
    HOME --> MENU
    HOME --> NOTIFY
    HOME --> DEVICES
    HOME --> ASSETS
    HOME --> RULES
    HOME --> DASHBOARDS
    DASHBOARDS --> DASH
    DASH --> GRID
    GRID --> WIDGET_WRAP
    WIDGET_WRAP --> WIDGET
    WIDGET --> DYN_WIDGET
```

### Component Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| Smart/Container | Connects to store, fetches data | Page components |
| Presentation | Receives inputs, emits outputs | Reusable UI components |
| Dynamic | Loaded at runtime via injector | Widgets |

### Component Lifecycle

```mermaid
sequenceDiagram
    participant Angular
    participant Component
    participant Store
    participant Service

    Angular->>Component: constructor (DI)
    Angular->>Component: ngOnInit
    Component->>Store: select(state)
    Store-->>Component: state$
    Component->>Service: fetchData()
    Service-->>Component: Observable<Data>

    loop User Interaction
        Component->>Store: dispatch(action)
        Store->>Store: reducer(state, action)
        Store-->>Component: updated state$
    end

    Angular->>Component: ngOnDestroy
    Component->>Component: unsubscribe all
```

## State Management

### Store Architecture

```mermaid
graph TB
    subgraph "NgRx Store"
        STATE[AppState]
    end

    subgraph "State Slices"
        LOAD[LoadState]
        AUTH[AuthState]
        SETTINGS[SettingsState]
        NOTIF[NotificationState]
    end

    subgraph "Components"
        COMP1[Component A]
        COMP2[Component B]
    end

    STATE --> LOAD
    STATE --> AUTH
    STATE --> SETTINGS
    STATE --> NOTIF

    COMP1 --> |"dispatch"| STATE
    COMP2 --> |"dispatch"| STATE
    STATE --> |"select"| COMP1
    STATE --> |"select"| COMP2
```

### State Structure

| Slice | Purpose | Key Properties |
|-------|---------|----------------|
| LoadState | HTTP loading indicators | isLoading, pendingRequests |
| AuthState | Authentication context | isAuthenticated, authUser, userDetails |
| SettingsState | User preferences | language, timezone, theme |
| NotificationState | Toast messages | notifications[] |

### NgRx Feature Modules

The application uses NgRx with feature state modules:

```mermaid
graph TB
    subgraph "NgRx Store"
        ROOT[Root State]
    end

    subgraph "Feature States"
        AUTH_F[auth feature]
        SETTINGS_F[settings feature]
        NOTIFY_F[notification feature]
    end

    subgraph "Lazy-Loaded States"
        DASH_F[dashboard feature]
        DEVICE_F[device feature]
    end

    ROOT --> AUTH_F
    ROOT --> SETTINGS_F
    ROOT --> NOTIFY_F
    ROOT -.->|lazy| DASH_F
    ROOT -.->|lazy| DEVICE_F
```

### State Organization

| Feature | Actions File | Reducer File | Effects File |
|---------|-------------|--------------|--------------|
| Auth | auth.actions.ts | auth.reducer.ts | auth.effects.ts |
| Settings | settings.actions.ts | settings.reducer.ts | - |
| Notification | notification.actions.ts | notification.reducer.ts | - |
| Load | load.actions.ts | load.reducer.ts | - |

### Auth State Details

```mermaid
graph TB
    subgraph "AuthState"
        IS_AUTH[isAuthenticated]
        USER_LOADED[isUserLoaded]
        AUTH_USER[authUser]
        USER_DETAILS[userDetails]
        PERMISSIONS[allowedDashboardIds]
        FEATURES[edgesSupportEnabled<br/>tbelEnabled<br/>mobileQrEnabled]
        SETTINGS[userSettings]
    end
```

### Data Flow Pattern

```mermaid
sequenceDiagram
    participant UI as Component
    participant Store as NgRx Store
    participant Effect as Effect
    participant API as HTTP Service

    UI->>Store: dispatch(LoadUsers)
    Store->>Effect: action stream
    Effect->>API: getUsers()
    API-->>Effect: User[]
    Effect->>Store: dispatch(LoadUsersSuccess)
    Store->>Store: reducer updates state
    Store-->>UI: select(users$)
    UI->>UI: render users
```

### Actions and Effects

| Action Type | Trigger | Effect |
|-------------|---------|--------|
| ActionAuthAuthenticated | Login success | Store user, navigate |
| ActionAuthUnauthenticated | Logout | Clear state, redirect |
| ActionAuthLoadUser | App init | Fetch user details |
| ActionLoadStart | HTTP request | Show loading |
| ActionLoadFinish | HTTP response | Hide loading |

## Routing

### Route Configuration

```mermaid
graph TB
    subgraph "Public Routes"
        LOGIN[/login]
        RESET[/login/resetPassword]
        MFA[/login/mfa]
    end

    subgraph "Protected Routes"
        HOME[/ home]
        ACCOUNT[/account]
        ADMIN[/admin]
        DEVICES[/device]
        ASSETS[/asset]
        DASHBOARDS[/dashboard]
        RULES[/rulechain]
    end

    subgraph "Guards"
        AUTH_GUARD[AuthGuard]
    end

    LOGIN --> HOME
    AUTH_GUARD --> HOME
    HOME --> ACCOUNT
    HOME --> ADMIN
    HOME --> DEVICES
    HOME --> ASSETS
    HOME --> DASHBOARDS
    HOME --> RULES
```

### Route Protection

```mermaid
sequenceDiagram
    participant User
    participant Router
    participant AuthGuard
    participant AuthService
    participant Store

    User->>Router: Navigate to /dashboard
    Router->>AuthGuard: canActivate?
    AuthGuard->>AuthService: isAuthenticated?
    AuthService->>Store: select(isAuthenticated)
    Store-->>AuthService: true/false

    alt Authenticated
        AuthGuard-->>Router: allow
        Router->>User: Show dashboard
    else Not Authenticated
        AuthGuard-->>Router: redirect
        Router->>User: Show login
    end
```

### Route Data

Routes include metadata for:
- Page title (i18n key)
- Breadcrumb configuration
- Required authorities
- Module visibility (public/private)

### Lazy Loading Pattern

```mermaid
graph LR
    subgraph "Initial Bundle"
        APP[App + Core + Shared]
    end

    subgraph "Lazy Chunks"
        LOGIN[login.chunk.js]
        HOME[home.chunk.js]
        DASH[dashboard.chunk.js]
    end

    APP --> |"loadChildren"| LOGIN
    APP --> |"loadChildren"| HOME
    HOME --> |"loadChildren"| DASH
```

## Authentication

### Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant LoginComponent
    participant AuthService
    participant API
    participant Store
    participant Router

    User->>LoginComponent: Enter credentials
    LoginComponent->>AuthService: login(email, password)
    AuthService->>API: POST /api/auth/login
    API-->>AuthService: JWT tokens

    AuthService->>AuthService: Store tokens (localStorage)
    AuthService->>Store: dispatch(Authenticated)
    Store->>Store: Update AuthState

    AuthService->>API: GET /api/auth/user
    API-->>AuthService: User details
    AuthService->>Store: dispatch(LoadUserSuccess)

    AuthService-->>LoginComponent: Success
    LoginComponent->>Router: Navigate to home
```

### Token Management

```mermaid
graph TB
    subgraph "Token Storage"
        ACCESS[jwt_token]
        ACCESS_EXP[jwt_token_expiration]
        REFRESH[refresh_token]
        REFRESH_EXP[refresh_token_expiration]
    end

    subgraph "Token Lifecycle"
        LOGIN[Login] --> STORE[Store Tokens]
        STORE --> USE[Use in Requests]
        USE --> CHECK{Near Expiry?}
        CHECK -->|Yes| REFRESH_REQ[Refresh Token]
        CHECK -->|No| USE
        REFRESH_REQ --> STORE
        LOGOUT[Logout] --> CLEAR[Clear Tokens]
    end
```

### Authority Levels

| Authority | Description | Access |
|-----------|-------------|--------|
| SYS_ADMIN | System administrator | All tenants |
| TENANT_ADMIN | Tenant administrator | Own tenant |
| CUSTOMER_USER | Customer user | Assigned resources |
| PUBLIC_USER | Anonymous | Public dashboards |

### Two-Factor Authentication

```mermaid
stateDiagram-v2
    [*] --> Login
    Login --> PreVerification: Credentials valid
    PreVerification --> MFAPrompt: 2FA required
    MFAPrompt --> Authenticated: Code valid
    MFAPrompt --> ForceMFASetup: Setup required
    ForceMFASetup --> Authenticated: Setup complete
    Authenticated --> [*]
```

## API Integration

### HTTP Service Architecture

```mermaid
graph TB
    subgraph "Components"
        COMP[Components]
    end

    subgraph "HTTP Services"
        USER_SVC[UserService]
        DEVICE_SVC[DeviceService]
        ASSET_SVC[AssetService]
        DASH_SVC[DashboardService]
        ENTITY_SVC[EntityService]
    end

    subgraph "HTTP Layer"
        CLIENT[HttpClient]
        INTERCEPT[Interceptors]
    end

    subgraph "Backend"
        API[REST API]
    end

    COMP --> USER_SVC
    COMP --> DEVICE_SVC
    COMP --> ASSET_SVC
    COMP --> DASH_SVC
    COMP --> ENTITY_SVC

    USER_SVC --> CLIENT
    DEVICE_SVC --> CLIENT
    ASSET_SVC --> CLIENT
    DASH_SVC --> CLIENT
    ENTITY_SVC --> CLIENT

    CLIENT --> INTERCEPT
    INTERCEPT --> API
```

### HTTP Interceptor Chain

```mermaid
sequenceDiagram
    participant Service
    participant GlobalInterceptor
    participant ConflictInterceptor
    participant HttpClient
    participant Backend

    Service->>GlobalInterceptor: request
    GlobalInterceptor->>GlobalInterceptor: Add JWT header
    GlobalInterceptor->>GlobalInterceptor: Start loading
    GlobalInterceptor->>ConflictInterceptor: request
    ConflictInterceptor->>HttpClient: request
    HttpClient->>Backend: HTTP request

    Backend-->>HttpClient: response
    HttpClient-->>ConflictInterceptor: response

    alt 409 Conflict
        ConflictInterceptor->>ConflictInterceptor: Show reload dialog
    end

    ConflictInterceptor-->>GlobalInterceptor: response

    alt 401 Unauthorized
        GlobalInterceptor->>GlobalInterceptor: Refresh token
    end

    GlobalInterceptor->>GlobalInterceptor: Stop loading
    GlobalInterceptor-->>Service: response
```

### Interceptor Responsibilities

| Interceptor | Responsibility |
|-------------|----------------|
| GlobalHttpInterceptor | JWT attachment, token refresh, loading state, error handling |
| EntityConflictInterceptor | Detect 409 conflicts, prompt reload |

### API Patterns

| Pattern | Example | Description |
|---------|---------|-------------|
| CRUD | `GET /api/device/{id}` | Standard entity operations |
| Pagination | `GET /api/devices?page=0&size=10` | PageLink-based results |
| Search | `POST /api/devices/query` | Entity queries |
| Telemetry | `GET /api/plugins/telemetry/...` | Time-series data |
| RPC | `POST /api/rpc/...` | Device commands |

## WebSocket Communication

### WebSocket Architecture

```mermaid
graph TB
    subgraph "Frontend"
        WIDGET[Widget Components]
        SUB_SVC[Subscription Service]
        WS_SVC[WebSocket Service]
    end

    subgraph "Connection"
        WS[WebSocket Connection]
    end

    subgraph "Backend"
        WS_API[/api/ws/plugins/telemetry]
    end

    WIDGET --> SUB_SVC
    SUB_SVC --> WS_SVC
    WS_SVC --> WS
    WS --> WS_API
```

### Subscription Flow

```mermaid
sequenceDiagram
    participant Widget
    participant SubscriptionService
    participant WebSocketService
    participant Backend

    Widget->>SubscriptionService: subscribe(datasource)
    SubscriptionService->>WebSocketService: createSubscription
    WebSocketService->>Backend: WS: Subscribe command
    Backend-->>WebSocketService: WS: Acknowledge

    loop Real-time Updates
        Backend-->>WebSocketService: WS: Data update
        WebSocketService-->>SubscriptionService: onData(update)
        SubscriptionService-->>Widget: dataUpdated$
        Widget->>Widget: Render new data
    end

    Widget->>SubscriptionService: unsubscribe
    SubscriptionService->>WebSocketService: removeSubscription
    WebSocketService->>Backend: WS: Unsubscribe command
```

### WebSocket Services

| Service | Purpose | Data Types |
|---------|---------|------------|
| TelemetryWebSocketService | Real-time telemetry | Attributes, timeseries, latest values |
| NotificationWebSocketService | Live notifications | Alarms, system events |

### Connection Management

| Feature | Behavior |
|---------|----------|
| Auto-reconnect | 2-second intervals |
| Idle timeout | 90 seconds, then reconnect |
| Batch commands | Up to 10 commands per message |
| Error recovery | Automatic resubscription |

## Widget System

### Widget Architecture

```mermaid
graph TB
    subgraph "Dashboard"
        DASH[DashboardComponent]
        GRID[Gridster Layout]
    end

    subgraph "Widget Container"
        CONTAINER[WidgetContainer]
        WIDGET_COMP[WidgetComponent]
        DYN[DynamicWidgetComponent]
    end

    subgraph "Widget Runtime"
        CONTEXT[WidgetContext]
        SUB[Subscription API]
        CTRL[Control API]
    end

    subgraph "Data Sources"
        DEVICE[Device Telemetry]
        ATTR[Attributes]
        ALARM[Alarms]
        ENTITY[Entity Data]
    end

    DASH --> GRID
    GRID --> CONTAINER
    CONTAINER --> WIDGET_COMP
    WIDGET_COMP --> DYN

    DYN --> CONTEXT
    CONTEXT --> SUB
    CONTEXT --> CTRL

    SUB --> DEVICE
    SUB --> ATTR
    SUB --> ALARM
    SUB --> ENTITY
```

### Widget Loading Flow

```mermaid
sequenceDiagram
    participant Dashboard
    participant WidgetService
    participant Injector
    participant DynamicWidget
    participant Subscription

    Dashboard->>WidgetService: loadWidget(config)
    WidgetService->>WidgetService: Resolve widget type
    WidgetService-->>Dashboard: Widget definition

    Dashboard->>Injector: createInjector(context)
    Injector-->>Dashboard: Custom injector

    Dashboard->>DynamicWidget: createComponent(injector)
    DynamicWidget->>DynamicWidget: ngOnInit

    DynamicWidget->>Subscription: subscribe(datasources)
    Subscription-->>DynamicWidget: data stream

    loop Updates
        Subscription-->>DynamicWidget: new data
        DynamicWidget->>DynamicWidget: render
    end
```

### Widget Context

| Property | Description |
|----------|-------------|
| subscriptionApi | Data subscription management |
| settings | Widget configuration |
| controlApi | Widget control methods |
| widget | Widget definition |
| parentDashboard | Dashboard reference |

### Widget Categories

| Type | Examples | Data Source |
|------|----------|-------------|
| Charts | Line, bar, pie | Timeseries |
| Gauges | Analog, digital, level | Latest values |
| Maps | OpenStreetMap, Google | Entity coordinates |
| Tables | Entities, alarms, timeseries | Query results |
| Controls | Buttons, switches, sliders | RPC commands |
| Cards | Value cards, HTML | Attributes/telemetry |

## Theming

### Theme Architecture

```mermaid
graph TB
    subgraph "Theme Definition"
        PRIMARY[Primary Palette]
        ACCENT[Accent Palette]
        WARN[Warn Palette]
    end

    subgraph "Theme Variants"
        LIGHT[Light Theme]
        DARK[Dark Theme]
    end

    subgraph "Component Styles"
        MATERIAL[Material Components]
        CUSTOM[Custom Components]
        WIDGETS[Widget Styles]
    end

    PRIMARY --> LIGHT
    PRIMARY --> DARK
    ACCENT --> LIGHT
    ACCENT --> DARK
    WARN --> LIGHT
    WARN --> DARK

    LIGHT --> MATERIAL
    LIGHT --> CUSTOM
    LIGHT --> WIDGETS
    DARK --> MATERIAL
    DARK --> CUSTOM
    DARK --> WIDGETS
```

### Theme Configuration

| Element | Light Theme | Dark Theme |
|---------|-------------|------------|
| Primary | Indigo | Dark Indigo |
| Accent | Deep Orange | Deep Orange |
| Background | #EEEEEE | #303030 |
| Text | Dark gray | Light gray |

### Styling Architecture

| Layer | Purpose | Scope |
|-------|---------|-------|
| Global styles | Typography, resets | Application-wide |
| Theme | Colors, palettes | Material components |
| Component SCSS | Component-specific | ViewEncapsulation |
| Widget styles | Dynamic styling | Widget runtime |

## Performance Patterns

### Optimization Strategies

```mermaid
graph TB
    subgraph "Bundle Optimization"
        LAZY[Lazy Loading]
        TREE[Tree Shaking]
        AOT[AOT Compilation]
    end

    subgraph "Runtime Optimization"
        ONPUSH[OnPush Detection]
        TRACK[trackBy Functions]
        UNSUB[Subscription Cleanup]
    end

    subgraph "Data Optimization"
        MEMO[Memoized Selectors]
        CACHE[Response Caching]
        BATCH[WebSocket Batching]
    end
```

### Change Detection

| Strategy | Use Case | Benefit |
|----------|----------|---------|
| Default | Simple components | Easy to implement |
| OnPush | List items, widgets | Reduced checks |
| Manual | Complex scenarios | Full control |

### Memory Management

```mermaid
sequenceDiagram
    participant Component
    participant Subscription
    participant Destroy$

    Component->>Destroy$: Create Subject
    Component->>Subscription: subscribe().pipe(takeUntil(destroy$))

    loop Component Active
        Subscription-->>Component: Data updates
    end

    Component->>Destroy$: next() + complete()
    Note over Subscription: All subscriptions cleaned up
```

## Error Handling

### Error Flow

```mermaid
graph TB
    subgraph "Error Sources"
        HTTP[HTTP Errors]
        WS[WebSocket Errors]
        APP[Application Errors]
    end

    subgraph "Error Handling"
        INTERCEPT[Interceptor]
        HANDLER[Global Handler]
    end

    subgraph "User Feedback"
        TOAST[Toast Notification]
        DIALOG[Error Dialog]
        CONSOLE[Console Log]
    end

    HTTP --> INTERCEPT
    INTERCEPT --> TOAST
    INTERCEPT --> DIALOG

    WS --> HANDLER
    APP --> HANDLER
    HANDLER --> TOAST
    HANDLER --> CONSOLE
```

### Error Types

| Code | Handling | User Feedback |
|------|----------|---------------|
| 401 | Token refresh or logout | Silent or login redirect |
| 403 | Display forbidden | Permission denied dialog |
| 404 | Display not found | Entity not found message |
| 409 | Conflict resolution | Reload entity prompt |
| 5xx | Retry logic | Server error toast |

## Build Configuration

### Build Pipeline

```mermaid
graph LR
    subgraph "Source"
        TS[TypeScript]
        SCSS[SCSS]
        HTML[Templates]
    end

    subgraph "Compilation"
        AOT[AOT Compiler]
        SASS[Sass Compiler]
        BUNDLE[Bundler]
    end

    subgraph "Output"
        JS[JavaScript Bundles]
        CSS[CSS Files]
        ASSETS[Static Assets]
    end

    TS --> AOT
    SCSS --> SASS
    HTML --> AOT

    AOT --> BUNDLE
    SASS --> BUNDLE
    BUNDLE --> JS
    BUNDLE --> CSS
    BUNDLE --> ASSETS
```

### Output Structure

| Bundle | Contents | Loading |
|--------|----------|---------|
| main.js | Core application | Immediate |
| vendor.js | Third-party libraries | Immediate |
| polyfills.js | Browser compatibility | Immediate |
| login.chunk.js | Login module | On navigation |
| home.chunk.js | Home module | On navigation |

## See Also

- [Widget System](./widget-system.md) - Widget framework details
- [REST API Overview](../06-api-layer/rest-api-overview.md) - Backend API
- [WebSocket Overview](../06-api-layer/websocket-overview.md) - Real-time protocol
- [Authentication](../06-api-layer/authentication.md) - Auth details
- [Subscription Model](../06-api-layer/subscription-model.md) - Data subscriptions
