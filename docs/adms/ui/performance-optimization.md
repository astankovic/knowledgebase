---
layout: default
title: Performance Optimization
parent: User Interface
grand_parent: ADMS Overview
nav_order: 5
---

# Performance Optimization

## Overview

Client-side performance optimization ensures ADMS user interfaces remain responsive even when displaying large datasets, processing frequent real-time updates, or rendering complex visualizations. Performance directly impacts operational effectiveness: operators making time-critical decisions during emergencies require instant UI responsiveness without delays, freezes, or visual glitches that could impede situational awareness.

Performance optimization addresses multiple bottlenecks: JavaScript execution time for application logic, DOM manipulation costs for rendering updates, network latency for data fetching, memory consumption for large datasets, and layout thrashing from excessive reflows. Optimization strategies balance competing concerns: aggressive caching improves speed but increases memory usage, code splitting reduces initial load but adds complexity, and virtualization improves rendering but complicates interaction handling.

Modern web performance APIs enable measurement and monitoring of real user performance. Navigation Timing API tracks page load metrics, Performance Observer API monitors long tasks blocking the main thread, and User Timing API measures custom application operations. Production monitoring aggregates metrics from all users, identifying performance regressions and guiding optimization priorities.

## Rendering Optimization

### Virtual Scrolling

Virtual scrolling (windowing) renders only visible list items rather than entire datasets, dramatically improving performance for long lists (thousands of alarms, equipment, measurements). The technique maintains a sliding window of rendered items, adding items entering the viewport while removing items leaving it. Scroll position calculations determine which items are visible, creating DOM elements on-demand.

Libraries like react-window and react-virtualized implement virtual scrolling with additional optimizations: overscan rendering (render items slightly outside viewport to prevent blank areas during fast scrolling), item size estimation (handle variable-height items), and scroll position persistence (restore scroll position after data updates).

Virtual scrolling reduces DOM node counts from tens of thousands to hundreds, eliminating browser rendering bottlenecks. However, it adds complexity: scrollbar sizing requires knowing total content height, accessibility features (screen readers) need special handling, and item selection must account for non-rendered items.

### Memoization

Memoization caches expensive computation results, returning cached values when inputs haven't changed. React provides memoization primitives: React.memo wraps components to prevent re-renders when props are unchanged, useMemo caches computed values, and useCallback caches function references to prevent prop changes triggering child re-renders.

Example: Computing alarm statistics (count by severity, by equipment type) from alarm lists is expensive with thousands of alarms. Memoizing statistics calculations with useMemo recomputes only when the alarm list changes, not on every render.

Memoization trades memory for speed: cached results consume memory, and comparison overhead adds cost. Apply memoization to expensive operations (complex calculations, large array transformations) triggered frequently. Avoid memoizing cheap operations where comparison costs exceed computation costs.

### Debouncing and Throttling

Debouncing delays executing functions until after a quiet period, useful for search inputs where executing queries on every keystroke wastes resources. Debouncing waits until user stops typing (300ms quiet) before executing search, reducing queries from dozens to one.

Throttling limits execution frequency, ensuring functions execute at most once per time period (100ms), regardless of invocation frequency. Throttling is ideal for scroll/resize handlers that fire continuously during user interactions. Throttled handlers execute periodically (10 times per second) rather than hundreds of times per second.

Implementation uses setTimeout/clearTimeout for debouncing and timestamps for throttling. Libraries like Lodash provide production-ready implementations handling edge cases like leading/trailing execution, maximum wait times, and cancellation.

## Data Management

### Pagination

Pagination loads large datasets incrementally, retrieving manageable chunks (50-100 items) rather than entire datasets. Offset-based pagination uses offset/limit parameters: load items 0-49, then 50-99, then 100-149. Cursor-based pagination uses opaque cursors pointing to specific positions in result sets, providing stable pagination when data changes frequently.

Infinite scroll implements pagination transparently: load first page initially, load next page automatically when user scrolls near bottom. This pattern feels seamless but complicates features like "scroll to top" and makes total item counts less visible.

Virtual scroll with pagination combines windowing (render only visible items) and lazy loading (fetch data as needed), handling effectively unlimited datasets. Implementation tracks which data pages are loaded, fetching pages containing visible items on-demand.

### Caching

Client-side caching stores frequently accessed data in memory or browser storage, reducing network requests and improving responsiveness. Cache strategies include:

- **Memory cache**: Store in JavaScript variables/Redux state, fastest access but volatile (lost on refresh)
- **LocalStorage**: Persist across sessions, survives page reloads but limited to 5-10MB and synchronous API
- **IndexedDB**: Large async storage (hundreds of MB) for structured data, complexity higher than LocalStorage
- **Service Workers**: Cache API responses offline, enabling progressive web app capabilities

Cache invalidation is the hard problem: determining when cached data is stale. Strategies include time-based expiration (cache for 5 minutes), version-based invalidation (cache includes version, invalidate on version change), and event-driven invalidation (server push notifications trigger cache clears).

### Differential Updates

Differential updates apply changes to datasets without replacing entire datasets, reducing network transfer and client-side processing. WebSocket messages include only changed equipment status rather than entire equipment lists. Client-side code updates affected items in-place, avoiding full list re-renders.

Immutable update patterns (Redux reducers) enable efficient change detection: React detects changes through reference equality (===) rather than deep equality, so immutable updates ensure components re-render only when data actually changes. Immutability libraries (Immer, Immutable.js) simplify writing immutable updates.

## Bundle Optimization

### Code Splitting

Code splitting divides application bundles into chunks loaded on-demand rather than loading everything upfront. Route-based splitting loads components when users navigate to routes. Component-based splitting defers loading heavy components (network diagram viewer) until they render.

Dynamic imports enable code splitting: `const NetworkDiagram = React.lazy(() => import('./NetworkDiagram'))`. Webpack bundles dynamically imported modules into separate chunks loaded when import() executes. Suspense components display loading indicators while chunks load.

Strategic code splitting reduces initial bundle size from 2MB+ to 200-300KB, improving startup time dramatically. However, excessive splitting creates many small chunks with overhead, and splitting at wrong boundaries causes chunks to load synchronously defeating the purpose.

### Tree Shaking

Tree shaking eliminates unused code during bundling, reducing bundle sizes by excluding library code never referenced. ES6 modules enable tree shaking: import/export are statically analyzable, unlike CommonJS require() which executes at runtime. Bundlers (Webpack, Rollup) analyze import graphs, marking used exports and eliminating unused code.

Effective tree shaking requires:
- Using ES6 module syntax (import/export) exclusively
- Importing only needed functions (`import { Button } from 'library'`) not entire libraries (`import * as Library`)
- Configuring production builds with optimization enabled
- Using libraries with tree-shake-friendly builds (many provide both CommonJS and ES modules)

### Compression

Compression reduces bundle sizes through encoding and minification. Minification removes whitespace, shortens variable names, and eliminates dead code. Gzip and Brotli compress text assets (JavaScript, CSS, HTML) to 20-30% of original sizes. Modern hosting enables compression automatically, but development builds should validate compression is working.

Bundle analysis tools (webpack-bundle-analyzer) visualize bundle composition, identifying large dependencies and code duplication. Analysis informs optimization: replacing large libraries with smaller alternatives, eliminating duplicate dependencies, or lazy-loading heavy features.

## Code Examples

### Virtual Scrolling Implementation

```typescript
import { FixedSizeList } from 'react-window';

interface Alarm {
  id: string;
  message: string;
  severity: string;
  timestamp: Date;
}

const VirtualizedAlarmList: React.FC<{ alarms: Alarm[] }> = ({ alarms }) => {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => {
    const alarm = alarms[index];
    return (
      <div style={style} className="alarm-item">
        <span className={`severity-${alarm.severity.toLowerCase()}`}>
          {alarm.severity}
        </span>
        <span>{alarm.message}</span>
        <span>{alarm.timestamp.toLocaleTimeString()}</span>
      </div>
    );
  };

  return (
    <FixedSizeList
      height={600}
      itemCount={alarms.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
};
```

### Memoized Computations

```typescript
const AlarmStatistics: React.FC<{ alarms: Alarm[] }> = ({ alarms }) => {
  // Memoize expensive statistics calculation
  const stats = useMemo(() => {
    const bySeverity = alarms.reduce((acc, alarm) => {
      acc[alarm.severity] = (acc[alarm.severity] || 0) + 1;
      return acc;
    }, {} as Record<string, number>);

    const byEquipment = alarms.reduce((acc, alarm) => {
      acc[alarm.equipmentType] = (acc[alarm.equipmentType] || 0) + 1;
      return acc;
    }, {} as Record<string, number>);

    return { bySeverity, byEquipment, total: alarms.length };
  }, [alarms]); // Recompute only when alarms array changes

  return (
    <div>
      <h3>Total Alarms: {stats.total}</h3>
      <div>
        <h4>By Severity</h4>
        {Object.entries(stats.bySeverity).map(([severity, count]) => (
          <div key={severity}>{severity}: {count}</div>
        ))}
      </div>
    </div>
  );
};
```

### Debounced Search

```typescript
const SearchInput: React.FC<{ onSearch: (query: string) => void }> = ({ onSearch }) => {
  const [query, setQuery] = useState('');

  // Debounce search execution
  const debouncedSearch = useMemo(
    () => debounce((value: string) => {
      onSearch(value);
    }, 300),
    [onSearch]
  );

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value);
    debouncedSearch(value);
  };

  return (
    <input
      type="text"
      value={query}
      onChange={handleChange}
      placeholder="Search equipment..."
    />
  );
};

// Debounce utility
function debounce<T extends (...args: any[]) => any>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: NodeJS.Timeout;
  return (...args: Parameters<T>) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
}
```

### React Performance Optimization

```typescript
// Memoize component to prevent unnecessary re-renders
const EquipmentItem = React.memo<{ equipment: Equipment; onSelect: (id: string) => void }>(
  ({ equipment, onSelect }) => {
    // Memoize callback to prevent prop changes
    const handleClick = useCallback(() => {
      onSelect(equipment.id);
    }, [equipment.id, onSelect]);

    return (
      <div className="equipment-item" onClick={handleClick}>
        <h3>{equipment.name}</h3>
        <p>Status: {equipment.status}</p>
        <p>Voltage: {equipment.voltage} kV</p>
      </div>
    );
  },
  // Custom comparison function for memo
  (prevProps, nextProps) => {
    return prevProps.equipment.id === nextProps.equipment.id &&
           prevProps.equipment.status === nextProps.equipment.status &&
           prevProps.equipment.voltage === nextProps.equipment.voltage;
  }
);
```

### Performance Monitoring

```typescript
class PerformanceMonitor {
  measureOperation(name: string, operation: () => void) {
    performance.mark(`${name}-start`);
    operation();
    performance.mark(`${name}-end`);

    performance.measure(name, `${name}-start`, `${name}-end`);

    const measure = performance.getEntriesByName(name)[0];
    console.log(`${name} took ${measure.duration.toFixed(2)}ms`);

    // Send to analytics
    this.reportMetric(name, measure.duration);

    // Cleanup
    performance.clearMarks(`${name}-start`);
    performance.clearMarks(`${name}-end`);
    performance.clearMeasures(name);
  }

  reportMetric(name: string, duration: number) {
    // Send to monitoring service (e.g., Google Analytics, Sentry)
    if (window.gtag) {
      window.gtag('event', 'timing_complete', {
        name: name,
        value: Math.round(duration),
        event_category: 'Performance'
      });
    }
  }

  observeLongTasks() {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > 50) { // Long task > 50ms
          console.warn(`Long task detected: ${entry.duration}ms`);
          this.reportMetric('long-task', entry.duration);
        }
      }
    });

    observer.observe({ entryTypes: ['longtask'] });
  }
}
```

## Best Practices

- Use virtual scrolling for lists exceeding 100 items to maintain rendering performance
- Apply memoization (React.memo, useMemo, useCallback) to expensive computations and prevent unnecessary re-renders
- Implement debouncing for user input handlers and throttling for frequent event handlers (scroll, resize)
- Load data incrementally using pagination rather than fetching entire datasets at once
- Split code by routes and features to reduce initial bundle size and improve startup time
- Monitor real user performance metrics (Core Web Vitals) to identify and prioritize optimization opportunities
- Use browser DevTools Performance profiler to identify bottlenecks in rendering and JavaScript execution
- Implement proper loading states and skeleton screens to improve perceived performance
- Set performance budgets (bundle size < 200KB, Time to Interactive < 3s) and monitor compliance
- Test performance with realistic datasets at production scale to identify issues before deployment
