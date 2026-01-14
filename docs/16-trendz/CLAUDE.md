# CLAUDE.md - Trendz Analytics Section

This file provides guidance for working with the Trendz Analytics documentation section.

## Section Purpose

This section documents Trendz Analytics:

- **Visualizations**: Chart types, view builder, data grouping
- **Calculations**: Calculated fields, aggregations, transformations
- **Predictions**: ML-based time series forecasting
- **Anomaly Detection**: Unsupervised anomaly scoring

## File Structure

```
16-trendz/
├── README.md                      # Trendz overview and capabilities
├── trendz-visualizations.md       # Chart types, view builder
├── trendz-calculations.md         # Calculated fields, predictions
└── trendz-anomaly-detection.md    # Anomaly models, scoring, alerting
```

## Writing Guidelines

### Audience

Data analysts and IoT engineers creating analytics dashboards. Assume familiarity with data visualization concepts but not necessarily with Trendz-specific patterns.

### Content Pattern

Trendz documents should include:

1. **Overview** - What the feature does
2. **Configuration** - Settings and options
3. **Examples** - Working examples with code
4. **Use Cases** - Industry applications
5. **Performance** - Optimization tips
6. **Pitfalls** - Common mistakes
7. **See Also** - Related documentation

### Analytics Documentation Pattern

For analytics features:

```markdown
## Feature Name

**Purpose**: What it analyzes

### Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| ... | ... | ... |

### Example

\`\`\`javascript
// Calculation example
var value = avg(Sensor.temperature);
return value * 1.8 + 32;
\`\`\`

### Use Cases

| Industry | Application |
|----------|-------------|
| ... | ... |

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| ... | ... |
```

### Terminology

- Use "View" for a Trendz visualization
- Use "Calculated Field" for custom transformations
- Use "Prediction" for ML-based forecasting
- Use "Anomaly Score" for deviation measurement
- Use "State" for condition-based duration tracking

### Diagrams

Use Mermaid diagrams to show:

- Data flow (`graph TB`)
- Analysis pipeline (`sequenceDiagram`)
- Prediction workflow (`sequenceDiagram`)
- Anomaly detection (`graph LR`)

### Technology-Agnostic Rule

Focus on analytics behavior, not implementation:

**DO**: "Trendz uses machine learning to predict future values based on historical patterns"
**DON'T**: "TrendzPredictionService calls Prophet.fit() via PyBridge connector"

**DO**: "Anomaly detection uses unsupervised clustering to identify unusual patterns"
**DON'T**: "AnomalyDetector implements IsolationForest with sklearn pipeline"

**DO**: "Calculated fields support JavaScript for custom transformations"
**DON'T**: "CalculatedFieldExecutor uses GraalJS with sandbox restrictions"

## Reference Sources

When updating this section, cross-reference:

- Trendz official documentation
- ThingsBoard Analytics documentation

## Related Sections

- `04-rule-engine/` - Data processing
- `02-core-concepts/data-model/` - Entity structure
- `09-security/` - Access control

## Common Tasks

### Documenting Visualizations

1. Show chart type options
2. Document data grouping
3. Explain aggregation functions
4. Include dashboard embedding

### Documenting Predictions

1. Explain available models
2. Document training data requirements
3. Show forecast configuration
4. Include accuracy metrics

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/16-trendz/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **iot-engineer** | `/iot-engineer` | IoT analytics patterns |
| **technical-writer** | `/technical-writer` | Clear analytics documentation |

### When to Use Each Skill

- **Documenting IoT analytics**: Use `/iot-engineer` for use cases
- **Writing calculation guides**: Use `/technical-writer` for clarity

## Key Trendz Concepts

When documenting Trendz, emphasize:

| Concept | Key Points |
|---------|------------|
| **Visualizations** | 8+ chart types with interactive builder |
| **Calculated Fields** | JavaScript/Python transformations |
| **Predictions** | ML models (Fourier, Prophet, ARIMA) |
| **Anomaly Detection** | Unsupervised anomaly scoring |
| **States** | Duration tracking across conditions |
| **Dashboard Embedding** | Trendz views in ThingsBoard dashboards |

## Visualization Types

| Type | Best For |
|------|----------|
| Line Chart | Time series trends |
| Bar Chart | Categorical comparisons |
| Pie Chart | Proportional breakdown |
| Heatmap | Density visualization |
| Scatter Plot | Correlation analysis |
| Table | Raw/aggregated data |
| Calendar | Date-based patterns |
| Card | Single KPI display |

## Prediction Models

| Model | Use Case |
|-------|----------|
| Fourier | Cyclic patterns |
| Prophet | Seasonal with holidays |
| ARIMA | Trend/seasonal data |
| Linear Regression | Simple trends |
| Custom Python | Advanced use cases |

## Common Pitfalls to Document

Ensure documentation covers these Trendz issues:

| Pitfall | Description |
|---------|-------------|
| Large dataset performance | Query timeouts on big data |
| Complex calculations | Batch processing needed |
| Prediction training data | Insufficient historical data |
| Anomaly false positives | Incorrect baseline period |
| Cache invalidation | Stale data after updates |
| Time zone issues | Incorrect time aggregation |

## Aggregation Functions

| Function | Description |
|----------|-------------|
| `avg()` | Average value |
| `sum()` | Total sum |
| `min()` | Minimum value |
| `max()` | Maximum value |
| `count()` | Count of values |
| `uniq()` | Unique value |
| `none()` | Raw value |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
