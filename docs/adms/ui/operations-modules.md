---
layout: default
title: Operations Modules
parent: User Interface
grand_parent: ADMS Overview
nav_order: 3
---

# Operations Modules

## Overview

Operations modules provide operator workstation functionality for monitoring and controlling electrical distribution networks in real-time. These modules implement core operational workflows including alarm management, switching order execution, measurement monitoring, and analytical application interaction. The user interface balances information density (operators need comprehensive situational awareness) with clarity (critical information must be immediately recognizable under time pressure).

Operational interfaces prioritize reliability and predictability. Operators work under stress during emergency conditions; interfaces must behave consistently, provide clear feedback for all actions, and prevent accidental operations through confirmations and interlocks. Design follows utility industry conventions where trained operators expect specific workflows, terminology, and visual patterns based on experience with SCADA systems and prior-generation ADMS platforms.

Real-time data integration ensures operators see current grid conditions with sub-second latency from field events to screen updates. Visual and audible indicators alert operators to condition changes requiring attention. Keyboard shortcuts and customizable quick-access panels enable experienced operators to work efficiently without mouse navigation for routine operations.

## Network Operations

### Switching Orders

Switching order management guides operators through multi-step equipment switching sequences for planned maintenance, load transfers, or service restoration. The system displays step-by-step instructions, validates each step before proceeding, and maintains audit trails of all operations. Visual workflows show switching sequence steps with current step highlighted, completed steps marked green, and pending steps in gray.

Each switching step specifies the equipment to operate (e.g., "Open Breaker 52-123"), expected equipment state after operation, safety tags to apply, and verification checks required. The system enforces sequential execution, preventing operators from executing step 5 before completing steps 1-4. Equipment control interfaces integrate directly, enabling single-click execution with confirmation dialogs describing the operation and expected outcome.

Switching order states include Draft (being prepared), Approved (ready for execution), In Progress (partially executed), Completed (all steps executed), and Canceled. State transitions trigger notifications to supervisors and logs for compliance auditing. Operators can pause execution, skip non-critical steps with justification, or abort sequences during emergencies.

### Tagging

Safety tagging workflow implements utility safety procedures where equipment scheduled for maintenance receives protective tags preventing inadvertent operation. Operators place logical tags in ADMS corresponding to physical tags at equipment locations. Tagged equipment blocks control operations with warnings explaining the tag reason and requiring supervisory override for emergencies.

Tag types include Clearance Tags (equipment de-energized for personnel safety), Hold Tags (equipment reserved for specific operation), and Restriction Tags (equipment operable only under specific conditions). Tags specify holder (person/team responsible), reason, placement time, and expected removal time. Tag management screens list all active tags with filtering by equipment, holder, or date range.

### Load Management

Load management modules enable operators to shed load during capacity shortages or execute demand response programs. Operators select load control zones, specify load reduction targets (MW), and execute shedding sequences. The system calculates customer impacts, prioritizes non-critical loads, and provides estimated restoration times.

Real-time feedback displays actual load reduction achieved versus targets, remaining load control capacity, and system frequency or voltage response. Automated load restoration procedures reclose feeders sequentially as capacity becomes available, avoiding synchronized inrush that could destabilize the grid.

## Monitoring and Analysis

### Alarm Management

Alarm management presents operators with current alarm conditions requiring attention. Alarms list in priority order (Critical, Major, Minor) with severity-based color coding. Each alarm includes equipment identification, alarm description, timestamp, current equipment status, and recommended operator actions. Alarm details expand to show related equipment, recent measurement trends, and historical alarm occurrences.

Operators acknowledge alarms to indicate awareness, suppressing audible annunciation. Alarm filtering controls hide acknowledged alarms, alarms for specific equipment types, or alarms below selected severity thresholds. Custom alarm views save filter configurations for different operational scenarios (normal operations vs storm restoration).

Alarm correlation groups related alarms into single logical events. A faulted transformer generates alarms for overcurrent, low voltage, and protection trip; correlation combines these into a single "Transformer Fault" alarm with subordinate alarms accessible through drill-down. This reduces alarm flooding during large-scale disturbances.

### Measurement Dashboard

Measurement dashboards display real-time telemetry in customizable layouts. Operators add widgets showing individual measurements as numeric displays, analog gauges (speedometer-style indicators), or trending charts. Dashboard grid layouts enable resizing and repositioning widgets to optimize information flow for specific operational roles.

Trending charts display historical measurements over configurable time ranges (last hour, last 24 hours, last week) with automatic Y-axis scaling or user-defined ranges. Multiple measurements overlay on single charts for comparison (e.g., voltage at three substations on same feeders). Zoom interactions focus on specific time periods, and cursor crosshairs show precise values at selected times.

Measurement quality indicators (color-coded borders) flag suspect or stale data. Operators hover over quality indicators to see quality explanations (communication failure, out-of-range, manually substituted). Manual data entry enables operators to input estimated values when telemetry is unavailable, with clear visual distinction from real measurements.

### Historical Trending

Historical trending retrieves archived measurements for analysis of past events, performance evaluation, or validation of operational decisions. Query interfaces specify measurement points, time ranges, aggregation intervals (raw values, 1-minute averages, hourly peaks), and export formats (CSV, Excel).

Trend queries return data tables and charts. Interactive charts enable panning across time ranges, zooming to specific periods, and toggling measurement series visibility. Statistical summaries include min, max, average, and standard deviation for selected periods. Comparison mode overlays multiple time periods to compare similar events (e.g., comparing last week's peak load to same weekday from previous year).

## Analytical Applications

### Power Flow Analysis

Power flow analysis executes network calculations determining voltage magnitudes, power flows, and equipment loading under current or planned conditions. Operators initiate power flow studies, selecting calculation parameters (consider distributed generation, model voltage regulator control, include line losses). Calculation status indicators show progress, with results displayed upon completion.

Results visualization includes:
- Voltage contour maps color-coding equipment by voltage (red for over-voltage, green for normal, yellow for under-voltage)
- Loading bars showing equipment utilization percentages (transformers, lines, cables)
- Tabular result listings sortable by voltage deviation, loading percentage, or equipment name
- Violation summaries highlighting equipment exceeding operational limits

Scenario comparison enables side-by-side display of multiple power flow results (current condition vs planned switching operation) highlighting changes. Difference maps show voltage and loading changes between scenarios.

### Outage Management

Outage management interface coordinates service restoration activities. Operators log customer outage reports, identify affected equipment through network tracing, dispatch crews to fault locations, and track restoration progress. Map-based interfaces display outage locations, crew positions, and restoration zones.

Customer impact analysis uses spatial joins between de-energized equipment and customer service locations to estimate affected customer counts. Priority indices (hospitals, fire stations, critical customers) escalate restoration activities. Estimated restoration times calculate based on crew travel times, typical repair durations, and switching sequence execution times.

### Fault Location

Fault location tools analyze protection relay data and measurement patterns to pinpoint fault locations along feeders. Operators input relay target information (phase relays operated, measured fault current, trip time), and the system calculates distance-to-fault using impedance-based methods. Map visualizations show calculated fault locations with confidence intervals.

Historical fault analysis compares current fault with previous faults on the same feeder, identifying recurring problem locations. Equipment maintenance recommendations generate automatically for repeatedly faulted equipment segments, feeding work management system integration.

## Code Examples

### Alarm List Component

```typescript
interface Alarm {
  id: string;
  equipmentId: string;
  equipmentName: string;
  severity: 'CRITICAL' | 'MAJOR' | 'MINOR';
  message: string;
  timestamp: Date;
  acknowledged: boolean;
}

const AlarmList: React.FC = () => {
  const alarms = useSelector(selectAlarms);
  const dispatch = useDispatch();
  const [filter, setFilter] = useState({ showAcknowledged: false, minSeverity: 'MINOR' });

  const filteredAlarms = alarms.filter(alarm => {
    if (!filter.showAcknowledged && alarm.acknowledged) return false;
    const severityOrder = { CRITICAL: 3, MAJOR: 2, MINOR: 1 };
    return severityOrder[alarm.severity] >= severityOrder[filter.minSeverity];
  });

  const handleAcknowledge = (alarmId: string) => {
    dispatch(acknowledgeAlarm(alarmId));
  };

  const getSeverityColor = (severity: string) => {
    switch (severity) {
      case 'CRITICAL': return '#d32f2f';
      case 'MAJOR': return '#f57c00';
      case 'MINOR': return '#fbc02d';
      default: return '#757575';
    }
  };

  return (
    <div className="alarm-list">
      <div className="alarm-filters">
        <label>
          <input
            type="checkbox"
            checked={filter.showAcknowledged}
            onChange={(e) => setFilter({ ...filter, showAcknowledged: e.target.checked })}
          />
          Show Acknowledged
        </label>
        <select
          value={filter.minSeverity}
          onChange={(e) => setFilter({ ...filter, minSeverity: e.target.value })}
        >
          <option value="MINOR">Minor+</option>
          <option value="MAJOR">Major+</option>
          <option value="CRITICAL">Critical Only</option>
        </select>
      </div>

      <div className="alarm-items">
        {filteredAlarms.map(alarm => (
          <div
            key={alarm.id}
            className="alarm-item"
            style={{ borderLeft: `4px solid ${getSeverityColor(alarm.severity)}` }}
          >
            <div className="alarm-header">
              <span className="alarm-severity">{alarm.severity}</span>
              <span className="alarm-time">{alarm.timestamp.toLocaleTimeString()}</span>
            </div>
            <div className="alarm-equipment">{alarm.equipmentName}</div>
            <div className="alarm-message">{alarm.message}</div>
            {!alarm.acknowledged && (
              <button onClick={() => handleAcknowledge(alarm.id)}>
                Acknowledge
              </button>
            )}
          </div>
        ))}
      </div>
    </div>
  );
};
```

### Measurement Trending Widget

```typescript
const MeasurementTrend: React.FC<{ measurementId: string }> = ({ measurementId }) => {
  const [data, setData] = useState<Array<{ time: Date; value: number }>>([]);
  const [timeRange, setTimeRange] = useState(3600000); // 1 hour in ms

  useEffect(() => {
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - timeRange);

    measurementApi.getHistory(measurementId, startTime, endTime)
      .then(history => setData(history));
  }, [measurementId, timeRange]);

  return (
    <div className="measurement-trend">
      <div className="trend-header">
        <h3>Voltage - Substation A</h3>
        <select
          value={timeRange}
          onChange={(e) => setTimeRange(Number(e.target.value))}
        >
          <option value={3600000}>Last Hour</option>
          <option value={86400000}>Last 24 Hours</option>
          <option value={604800000}>Last Week</option>
        </select>
      </div>
      <LineChart width={400} height={200} data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="time" tickFormatter={(time) => new Date(time).toLocaleTimeString()} />
        <YAxis domain={[11.5, 13.5]} />
        <Tooltip labelFormatter={(time) => new Date(time).toLocaleString()} />
        <Line type="monotone" dataKey="value" stroke="#8884d8" dot={false} />
      </LineChart>
    </div>
  );
};
```

## Best Practices

- Prioritize alarm management clarity with severity-based color coding and priority-ordered lists
- Implement comprehensive audit logging of all control operations with operator identification and timestamps
- Provide multi-step confirmations for safety-critical operations with clear descriptions of actions and consequences
- Enable keyboard shortcuts for frequently-used operations to improve operator efficiency
- Display equipment status with consistent color schemes across all modules (green=normal, red=alarm, gray=out-of-service)
- Implement role-based UI hiding irrelevant features and preventing unauthorized operations gracefully
- Provide real-time status feedback for long-running operations (calculations, switching sequences) with progress indicators
- Design for high-stress operational scenarios with large, clear controls and minimal cognitive load
- Test interfaces with actual operators to validate workflows match operational procedures
- Maintain consistency with existing utility control systems to leverage operator training and muscle memory
