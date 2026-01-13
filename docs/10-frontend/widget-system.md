# Widget System

## Overview

ThingsBoard's widget system provides a flexible, extensible framework for visualizing IoT data. Widgets are self-contained components that receive data through subscriptions and render interactive visualizations. The architecture supports built-in widgets, custom widgets, and third-party integrations through a well-defined API. This system enables dashboards to display real-time telemetry, historical trends, alarms, and device controls.

## Key Behaviors

1. **Type-Based Classification**: Widgets are categorized by data type (timeseries, latest, RPC, alarm, static).

2. **Subscription-Driven Data**: Widgets receive data through subscription callbacks rather than polling.

3. **Datasource Abstraction**: Widgets are decoupled from specific data sources through entity aliases.

4. **Action System**: Widgets can trigger navigation, commands, and custom behaviors.

5. **Dynamic Configuration**: Widget settings are defined through JSON schemas and dynamic forms.

6. **Lifecycle Management**: Widgets have well-defined initialization, update, and destruction phases.

## Widget Types

### Type Classification

```mermaid
graph TB
    subgraph "Widget Types"
        TS[timeseries<br/>Historical data visualization]
        LATEST[latest<br/>Current value display]
        RPC[rpc<br/>Device control]
        ALARM[alarm<br/>Alert monitoring]
        STATIC[static<br/>Static content]
    end

    subgraph "Example Widgets"
        TS --> CHART[Line Charts<br/>Bar Charts<br/>Area Charts]
        LATEST --> CARDS[Value Cards<br/>Gauges<br/>Indicators]
        RPC --> CONTROLS[Buttons<br/>Switches<br/>Sliders]
        ALARM --> TABLES[Alarm Tables<br/>Alert Lists]
        STATIC --> HTML[HTML Cards<br/>Images<br/>Markdown]
    end
```

### Type Characteristics

| Type | Data Pattern | Time Window | Use Case |
|------|--------------|-------------|----------|
| timeseries | Historical + streaming | Required | Trends, charts, graphs |
| latest | Most recent values | Not used | Current state, gauges |
| rpc | Command/response | Not used | Device control |
| alarm | Alarm events | Optional | Alert monitoring |
| static | None | Not used | Labels, images, info |

## Widget Bundles

### Bundle Organization

```mermaid
graph TB
    subgraph "System Bundles"
        CHARTS[Charts Bundle]
        GAUGES[Analogue Gauges]
        DIGITAL[Digital Gauges]
        MAPS[Maps Bundle]
        CARDS[Cards Bundle]
        TABLES[Tables Bundle]
        CONTROL[Control Widgets]
    end

    subgraph "Tenant Bundles"
        CUSTOM[Custom Bundle A]
        INDUSTRY[Industry-Specific]
    end

    subgraph "Widgets"
        CHARTS --> W1[Line Chart]
        CHARTS --> W2[Bar Chart]
        CHARTS --> W3[Pie Chart]
        GAUGES --> W4[Radial Gauge]
        GAUGES --> W5[Linear Gauge]
    end
```

### Bundle Structure

| Field | Description |
|-------|-------------|
| id | Unique identifier |
| tenantId | Owner (system or tenant) |
| alias | Unique string reference |
| title | Display name |
| image | Bundle preview image |
| description | Purpose documentation |
| order | Display sequence |
| widgets | Collection of widget types |

## Widget Configuration

### Configuration Hierarchy

```mermaid
graph TB
    subgraph "Configuration Layers"
        DEFAULT[Widget Type Defaults]
        BUNDLE[Bundle Template]
        INSTANCE[Dashboard Instance]
    end

    subgraph "Final Config"
        MERGED[Merged Configuration]
    end

    DEFAULT --> MERGED
    BUNDLE --> MERGED
    INSTANCE --> MERGED
```

### Configuration Structure

```mermaid
graph TB
    subgraph "WidgetConfig"
        subgraph "Display"
            TITLE[Title & Icon]
            STYLE[Styling]
            LAYOUT[Layout Options]
        end

        subgraph "Data"
            DS[Datasources]
            TW[Time Window]
            UNITS[Units & Decimals]
        end

        subgraph "Behavior"
            ACTIONS[Actions]
            SETTINGS[Widget Settings]
        end
    end
```

### Configuration Properties

| Category | Properties | Purpose |
|----------|------------|---------|
| Display | title, titleFont, titleColor, showTitle | Header appearance |
| Styling | widgetStyle, backgroundColor, padding, borderRadius | Visual appearance |
| Layout | resizable, preserveAspectRatio, mobileHide, mobileHeight | Sizing behavior |
| Data | datasources, timewindow, units, decimals | Data binding |
| Behavior | actions, settings, enableFullscreen | Interactivity |

## Data Sources

### Datasource Types

```mermaid
graph TB
    subgraph "Datasource Types"
        DEVICE[device<br/>Single device]
        ENTITY[entity<br/>Entity alias]
        FUNCTION[function<br/>Calculated data]
        ENTITY_COUNT[entityCount<br/>Entity count]
        ALARM_COUNT[alarmCount<br/>Alarm count]
    end

    subgraph "Resolution"
        DEVICE --> DEV_ID[Device ID]
        ENTITY --> ALIAS[Entity Alias → Filter]
        FUNCTION --> CALC[JavaScript Function]
    end
```

### Datasource Structure

```mermaid
graph TB
    subgraph "Datasource"
        TYPE[type]
        NAME[name]
        ALIAS[aliasName]
        KEYS[dataKeys]
        LATEST[latestDataKeys]
        FILTER[entityFilter]
        PAGE[pageLink]
    end

    subgraph "DataKey"
        KEY_NAME[name]
        KEY_TYPE[type]
        LABEL[label]
        COLOR[color]
        AGG[aggregationType]
        FUNC[funcBody]
        POST[postFuncBody]
    end

    KEYS --> KEY_NAME
    KEYS --> KEY_TYPE
    KEYS --> LABEL
```

### Data Key Types

| Type | Description | Example |
|------|-------------|---------|
| timeseries | Historical time-series | temperature, humidity |
| attribute | Entity attribute | serialNumber, firmware |
| function | Calculated value | avg(temp1, temp2) |
| constant | Static value | threshold = 100 |

### Aggregation Types

| Aggregation | Description |
|-------------|-------------|
| NONE | Raw data points |
| MIN | Minimum value in interval |
| MAX | Maximum value in interval |
| AVG | Average value in interval |
| SUM | Sum of values in interval |
| COUNT | Count of data points |

## Widget Actions

### Action Architecture

```mermaid
graph TB
    subgraph "Action Sources"
        HEADER[Header Button]
        ROW[Row Click]
        CELL[Cell Click]
        MARKER[Map Marker]
        CUSTOM[Custom Trigger]
    end

    subgraph "Action Types"
        NAV[Navigation Actions]
        CMD[Command Actions]
        CUSTOM_ACT[Custom Actions]
    end

    subgraph "Navigation"
        DASH_STATE[Open Dashboard State]
        UPDATE_STATE[Update State]
        OPEN_DASH[Open Dashboard]
        OPEN_URL[Open URL]
    end

    subgraph "Commands"
        RPC_CMD[RPC Command]
        MOBILE[Mobile Action]
    end

    HEADER --> NAV
    ROW --> NAV
    CELL --> NAV
    MARKER --> NAV

    NAV --> DASH_STATE
    NAV --> UPDATE_STATE
    NAV --> OPEN_DASH
    NAV --> OPEN_URL

    CMD --> RPC_CMD
    CMD --> MOBILE

    CUSTOM_ACT --> JS[Custom JavaScript]
```

### Action Types

| Type | Description | Target |
|------|-------------|--------|
| openDashboardState | Navigate to state | State ID |
| updateDashboardState | Update parameters | State params |
| openDashboard | Open dashboard | Dashboard ID |
| openURL | External link | URL |
| custom | JavaScript execution | Function body |
| mobileAction | Mobile-specific | Camera, GPS, etc. |

### Mobile Actions

| Action | Description |
|--------|-------------|
| takePictureFromGallery | Select from gallery |
| takePhoto | Capture photo |
| mapDirection | Navigation directions |
| mapLocation | Show on map |
| scanQrCode | QR code scanner |
| makePhoneCall | Initiate call |
| getLocation | Get GPS location |
| takeScreenshot | Capture screen |

### Action Configuration

```mermaid
graph TB
    subgraph "Action Definition"
        NAME[name]
        TYPE[type]
        ICON[icon]
        TARGET[targetDashboardId]
        STATE[targetDashboardStateId]
        OPTS[Options]
    end

    subgraph "Options"
        NEW_TAB[openNewBrowserTab]
        POPOVER[openInPopover]
        SET_ENTITY[setEntityId]
        DIALOG[dialogTitle/Width/Height]
    end

    OPTS --> NEW_TAB
    OPTS --> POPOVER
    OPTS --> SET_ENTITY
    OPTS --> DIALOG
```

## Widget Subscription

### Subscription Architecture

```mermaid
graph TB
    subgraph "Widget"
        WIDGET[Widget Component]
    end

    subgraph "Subscription Layer"
        SUB[IWidgetSubscription]
        CALLBACKS[Callbacks]
    end

    subgraph "Data Layer"
        WS[WebSocket Service]
        HTTP[HTTP Service]
    end

    subgraph "Backend"
        SERVER[ThingsBoard Server]
    end

    WIDGET --> SUB
    SUB --> CALLBACKS
    SUB --> WS
    SUB --> HTTP
    WS --> SERVER
    HTTP --> SERVER
```

### Subscription Flow

```mermaid
sequenceDiagram
    participant Widget
    participant Subscription
    participant WebSocket
    participant Server

    Widget->>Subscription: subscribe()
    Subscription->>WebSocket: createSubscription(datasources)
    WebSocket->>Server: Subscribe command

    loop Data Updates
        Server-->>WebSocket: Data update
        WebSocket-->>Subscription: onData(data)
        Subscription-->>Widget: onDataUpdated callback
        Widget->>Widget: Render new data
    end

    Widget->>Subscription: destroy()
    Subscription->>WebSocket: removeSubscription
    WebSocket->>Server: Unsubscribe command
```

### Subscription Callbacks

| Callback | Trigger | Purpose |
|----------|---------|---------|
| onDataUpdated | New data received | Update visualization |
| onLatestDataUpdated | Latest values changed | Update current state |
| onDataUpdateError | Data fetch failed | Show error state |
| dataLoading | Data fetch started | Show loading state |
| legendDataUpdated | Legend changed | Update legend |
| timeWindowUpdated | Time window changed | Adjust display |
| rpcStateChanged | RPC state changed | Update control state |
| onRpcSuccess | RPC succeeded | Show success feedback |
| onRpcFailed | RPC failed | Show error feedback |

## Widget Context API

### Context Structure

```mermaid
graph TB
    subgraph "WidgetContext"
        subgraph "References"
            DASH[dashboard]
            WIDGET[widget]
            SUB[subscriptions]
        end

        subgraph "Configuration"
            CONFIG[widgetConfig]
            SETTINGS[settings]
            UNITS[units]
            DECIMALS[decimals]
        end

        subgraph "APIs"
            SUB_API[subscriptionApi]
            ACTION_API[actionsApi]
            CTRL_API[controlApi]
            ALIAS_CTRL[aliasController]
            STATE_CTRL[stateController]
        end

        subgraph "Services"
            DEVICE_SVC[deviceService]
            ASSET_SVC[assetService]
            ENTITY_SVC[entityService]
            TELEMETRY_SVC[telemetryWsService]
            HTTP[http]
            DIALOG[dialogs]
        end

        subgraph "Data"
            DATA[data]
            LATEST[latestData]
            DS[datasources]
            TW[timeWindow]
        end

        subgraph "Utilities"
            UTILS[utils]
            TOAST[showToast methods]
        end
    end
```

### Available Services (40+ Injected)

The WidgetContext provides access to over 40 platform services:

**Entity Services:**
| Service | Purpose |
|---------|---------|
| deviceService | Device CRUD operations |
| assetService | Asset CRUD operations |
| entityService | Generic entity operations |
| entityRelationService | Entity relationships |
| entityGroupService | Entity group management |
| customerService | Customer operations |
| userService | User management |
| tenantService | Tenant operations |

**Data Services:**
| Service | Purpose |
|---------|---------|
| attributeService | Attribute read/write |
| telemetryWsService | WebSocket telemetry |
| alarmService | Alarm operations |
| deviceProfileService | Device profiles |
| assetProfileService | Asset profiles |

**Dashboard Services:**
| Service | Purpose |
|---------|---------|
| dashboardService | Dashboard operations |
| dashboardUtilsService | Dashboard utilities |
| widgetService | Widget definitions |
| widgetComponentService | Widget rendering |

**Platform Services:**
| Service | Purpose |
|---------|---------|
| authService | Authentication info |
| http | Raw HTTP client |
| dialogs | Dialog popups |
| translate | Internationalization (i18n) |
| router | Navigation |
| date | Date utilities |
| sanitizer | DOM sanitizer |
| ngZone | Angular zone |
| store | NgRx store access |
| rxjs | RxJS utilities |

**UI Services:**
| Service | Purpose |
|---------|---------|
| importExport | Import/export functionality |
| raf | Request animation frame |
| cd | Change detection |
| renderer | DOM renderer |

**Specialized Services:**
| Service | Purpose |
|---------|---------|
| ruleChainService | Rule chain operations |
| entityViewService | Entity view management |
| notificationService | Platform notifications |
| resourceService | Resource management |
| otaPackageService | OTA package management |

### Utility Functions

| Function | Description |
|----------|-------------|
| formatValue(value, decimals, units) | Format numeric value |
| getEntityDetailsPageURL(id, type) | Get entity page URL |

### Display Methods

| Method | Description |
|--------|-------------|
| showSuccessToast(message) | Green success notification |
| showErrorToast(message) | Red error notification |
| showToast(type, message, duration) | Custom toast |

## Widget Lifecycle

### Lifecycle Phases

```mermaid
stateDiagram-v2
    [*] --> TypeResolution
    TypeResolution --> ConfigParsing
    ConfigParsing --> SubscriptionCreation
    SubscriptionCreation --> ComponentCreation
    ComponentCreation --> ServiceInjection
    ServiceInjection --> OnInit
    OnInit --> Active

    Active --> DataUpdate: New data
    DataUpdate --> Active

    Active --> TimeWindowChange: Time change
    TimeWindowChange --> Active

    Active --> Resize: Size change
    Resize --> Active

    Active --> OnDestroy: Removed
    OnDestroy --> [*]
```

### Lifecycle Callbacks

```mermaid
sequenceDiagram
    participant Framework
    participant Widget
    participant Subscription

    Framework->>Widget: constructor(ctx)
    Framework->>Widget: onInit(ctx)
    Widget->>Subscription: subscribe()

    loop Active Phase
        Subscription-->>Widget: onDataUpdated(ctx)
        Framework-->>Widget: onResize(ctx)
    end

    Framework->>Widget: onDestroy(ctx)
    Widget->>Subscription: destroy()
```

### Controller Script Pattern

```javascript
// Initialization
self.onInit = function(ctx) {
    // Setup widget
    // Initialize state
    // Configure handlers
}

// Data handling
self.onDataUpdated = function(ctx) {
    // Process new data
    // Update visualization
}

// Layout handling
self.onResize = function(ctx) {
    // Adjust to new size
}

// RPC handling
self.onRpcSuccess = function(ctx, result) {
    // Handle success
}

self.onRpcFailed = function(ctx, error) {
    // Handle failure
}

// Cleanup
self.onDestroy = function(ctx) {
    // Release resources
    // Cancel subscriptions
}
```

## Custom Widget Development

### Widget Type Definition

```mermaid
graph TB
    subgraph "WidgetTypeDescriptor"
        TYPE[type: widgetType]
        RESOURCES[resources: URL[]]
        TEMPLATE[templateHtml]
        CSS[templateCss]
        CONTROLLER[controllerScript]
        SETTINGS_FORM[settingsForm]
        DEFAULT_CONFIG[defaultConfig]
        SIZE[sizeX, sizeY]
    end
```

### Custom Widget Structure

| Component | Purpose | Format |
|-----------|---------|--------|
| templateHtml | Widget HTML template | HTML string |
| templateCss | Widget styles | CSS string |
| controllerScript | Widget logic | JavaScript |
| settingsForm | Settings schema | JSON schema |
| resources | External dependencies | URL array |
| defaultConfig | Default configuration | JSON |

### Development Flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Editor as Widget Editor
    participant Preview as Preview
    participant Bundle as Widget Bundle

    Dev->>Editor: Write HTML template
    Dev->>Editor: Write CSS styles
    Dev->>Editor: Write controller script
    Dev->>Editor: Define settings form

    Editor->>Preview: Render preview
    Preview-->>Dev: Visual feedback

    Dev->>Editor: Configure defaults
    Dev->>Bundle: Save to bundle

    Bundle-->>Bundle: Available in library
```

### Widget Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| useCustomDatasources | Custom datasource support | false |
| maxDatasources | Maximum datasources | unlimited |
| maxDataKeys | Maximum data keys | unlimited |
| singleEntity | Single entity mode | false |
| stateData | Use dashboard state | false |
| hasBasicMode | Basic config UI | false |

## Value Processing

### Processing Pipeline

```mermaid
graph LR
    subgraph "Data Flow"
        RAW[Raw Value]
        FUNC[Function Processing]
        POST[Post-Processing]
        FORMAT[Formatting]
        DISPLAY[Display]
    end

    RAW --> FUNC
    FUNC --> POST
    POST --> FORMAT
    FORMAT --> DISPLAY
```

### Value Processors

| Processor | Purpose |
|-----------|---------|
| SimpleValueFormatProcessor | Basic formatting |
| UnitConverterValueFormatProcessor | Unit conversion |

### Color Processors

```mermaid
graph TB
    subgraph "Color Processors"
        CONST[ConstantColor<br/>Static color]
        FUNC[FunctionColor<br/>Calculated]
        RANGE[RangeColor<br/>Value ranges]
        GRAD[GradientColor<br/>Continuous]
    end

    subgraph "Examples"
        CONST --> C1["#FF0000"]
        FUNC --> C2["value > 50 ? 'red' : 'green'"]
        RANGE --> C3["0-50: green, 50-80: yellow, 80+: red"]
        GRAD --> C4["blue → red gradient"]
    end
```

### Date Formatters

| Formatter | Output Example |
|-----------|----------------|
| SimpleDateFormat | "2024-01-15 10:30:00" |
| LastUpdateAgo | "5 minutes ago" |
| AutoDateFormat | Context-aware format |

## Widget Settings Schema

### Dynamic Form Schema

```mermaid
graph TB
    subgraph "Form Property"
        KEY[key: string]
        LABEL[label: string]
        TYPE[type: string/number/boolean]
        REQ[required: boolean]
        VALID[validators]
        PROPS[properties: nested]
    end
```

### Property Types

| Type | Description | UI Control |
|------|-------------|------------|
| string | Text value | Text input |
| number | Numeric value | Number input |
| boolean | True/false | Checkbox/toggle |
| array | List of items | Array editor |
| object | Nested object | Nested form |
| color | Color value | Color picker |
| image | Image reference | Image selector |
| entity | Entity reference | Entity selector |

### Settings Form Example

```json
{
  "schema": {
    "title": "Widget Settings",
    "properties": {
      "showTitle": {
        "type": "boolean",
        "default": true
      },
      "titleColor": {
        "type": "string",
        "format": "color"
      },
      "decimals": {
        "type": "number",
        "minimum": 0,
        "maximum": 10
      }
    }
  }
}
```

## Best Practices

### Performance

| Practice | Benefit |
|----------|---------|
| Use OnPush change detection | Reduce render cycles |
| Unsubscribe in onDestroy | Prevent memory leaks |
| Use latest data optimization | Reduce data transfer |
| Implement pagination | Handle large datasets |

### Data Handling

| Practice | Benefit |
|----------|---------|
| Use entity aliases | Decouple from specific entities |
| Define aggregation | Reduce data volume |
| Set appropriate time window | Balance detail vs. performance |
| Handle empty states | Improve UX |

### Action Design

| Practice | Benefit |
|----------|---------|
| Use dashboard states | Maintain context |
| Provide feedback | User knows action worked |
| Handle errors gracefully | Don't break dashboard |
| Use confirmation for destructive actions | Prevent accidents |

## Common Widget Patterns

### Value Card Pattern

```mermaid
graph TB
    subgraph "Value Card"
        LABEL[Label]
        VALUE[Current Value]
        UNIT[Unit]
        TREND[Trend Indicator]
        TIMESTAMP[Last Update]
    end
```

### Chart Pattern

```mermaid
graph TB
    subgraph "Time Series Chart"
        LEGEND[Legend]
        AXIS_X[Time Axis]
        AXIS_Y[Value Axis]
        SERIES[Data Series]
        TOOLTIP[Hover Tooltip]
    end
```

### Table Pattern

```mermaid
graph TB
    subgraph "Data Table"
        HEADER[Column Headers]
        ROWS[Data Rows]
        PAGINATION[Pagination]
        ACTIONS[Row Actions]
        SORTING[Sort Controls]
    end
```

### Control Pattern

```mermaid
graph TB
    subgraph "Control Widget"
        LABEL[Control Label]
        INPUT[Input Control]
        STATUS[Connection Status]
        FEEDBACK[Action Feedback]
    end
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No data displayed | Subscription not active | Check datasource config |
| Data updates stop | WebSocket disconnected | Check connection status |
| Actions don't work | Missing entity context | Verify entity alias |
| Widget won't resize | CSS constraints | Check style settings |
| Slow performance | Too many data points | Enable aggregation |

### Debugging Tools

| Tool | Purpose |
|------|---------|
| Browser DevTools | JavaScript errors |
| Network tab | API requests |
| Widget debug mode | Data inspection |
| Console logging | Custom debugging |

## See Also

- [Angular Architecture](./angular-architecture.md) - Frontend framework
- [WebSocket Overview](../06-api-layer/websocket-overview.md) - Real-time data
- [Subscription Model](../06-api-layer/subscription-model.md) - Data subscriptions
- [REST API Overview](../06-api-layer/rest-api-overview.md) - API integration
