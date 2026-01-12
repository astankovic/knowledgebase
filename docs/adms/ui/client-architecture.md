---
layout: default
title: Client Architecture
parent: User Interface
grand_parent: ADMS Overview
nav_order: 1
---

# Client Architecture

## Overview

ADMS client architecture leverages modern web technologies to deliver rich, responsive user experiences while maintaining clean separation of concerns, testability, and maintainability. The architecture employs single-page application (SPA) patterns where the client application loads once and dynamically updates content without full page reloads, providing desktop-application-like responsiveness with web-based deployment simplicity.

The client stack comprises multiple layers: presentation components rendering UI elements, state management coordinating application data and logic, service layers abstracting backend API communication, and real-time data streams delivering live updates. Component-based architectures (React, Angular, Vue.js) promote code reuse, enable parallel development, and facilitate testing through component isolation.

Modern JavaScript/TypeScript development toolchains compile, bundle, and optimize code for production deployment. Module bundlers (Webpack, Vite) perform tree-shaking to eliminate unused code, code-splitting to load features on-demand, and minification to reduce download sizes. Development servers provide hot-module replacement for instant feedback during development, accelerating developer productivity.

## Technology Stack

### Frontend Framework

ADMS clients use React or Angular as primary UI frameworks, both providing component-based development, declarative rendering, and extensive ecosystems. React's virtual DOM and unidirectional data flow simplify reasoning about application behavior. React hooks enable functional components with stateful logic. JSX syntax blends JavaScript with HTML-like markup.

Angular provides a comprehensive framework with built-in routing, HTTP client, forms handling, and dependency injection. TypeScript is required (not optional), enforcing type safety. Angular's opinionated structure suits large enterprise applications with multiple teams. RxJS reactive programming handles asynchronous operations and real-time data streams elegantly.

Component libraries (Material-UI for React, Angular Material) provide pre-built UI components following Material Design principles: buttons, forms, modals, data tables, and navigation elements. Custom utility-specific components extend base libraries for specialized visualizations like network diagrams, measurement gauges, and alarm lists.

### State Management

Complex applications require centralized state management to coordinate data across components. Redux (React) and NgRx (Angular) implement Flux/Redux patterns with unidirectional data flow: actions describe state changes, reducers apply changes immutably, and stores notify subscribed components of updates.

State management architectures separate concerns:
- **Application State**: Current user session, authentication tokens, user preferences
- **Domain State**: Equipment data, measurements, alarms cached from backend
- **UI State**: Modal visibility, selected tabs, form input values

Middleware intercepts actions for logging, analytics, asynchronous API calls, and WebSocket message handling. Selectors compute derived state (filtered lists, aggregated statistics) with memoization to avoid unnecessary recalculation.

### UI Component Library

Material-UI (React) and Angular Material provide consistent, accessible UI components. Benefits include:
- Consistent visual design across application
- Accessibility features (ARIA labels, keyboard navigation)
- Theming support for light/dark modes and utility branding
- Responsive layouts adapting to screen sizes
- Pre-built complex components (data tables with sorting/filtering/pagination)

Custom components extend base libraries for ADMS-specific needs: network diagram viewers, real-time measurement displays, alarm management widgets. Component APIs follow framework conventions for props/inputs and events/outputs.

## Application Architecture

### Single Page Application (SPA)

SPA architecture loads a single HTML page with a JavaScript bundle that dynamically updates content based on user interactions and routing. Client-side routing (React Router, Angular Router) changes displayed components without server round-trips, providing instant navigation.

Initial page load includes minimal HTML, CSS, and JavaScript bootstrap. The application shell loads first, displaying loading indicators while fetching required data. Code-splitting divides the application into chunks loaded on-demand: the Operations Workspace code loads when users navigate to operations, not on initial page load.

Benefits include smooth user experience without page flashes, reduced server load (serving static files only), and ability to work offline with cached data. Drawbacks include larger initial bundle sizes, SEO challenges (mitigated with server-side rendering), and complexity in state management.

### Module Federation

Micro-frontend architectures using Module Federation (Webpack 5) enable independent development and deployment of application modules. The shell application loads feature modules dynamically at runtime. The Operations module, Engineering module, and Administration module can be developed by separate teams, deployed independently, and integrated at runtime.

This approach scales development across large teams, enables technology diversity (different React versions in different modules), and facilitates gradual upgrades. However, it adds complexity in shared dependencies, version coordination, and runtime integration.

### Lazy Loading

Lazy loading defers loading of features until needed, reducing initial bundle sizes and improving startup performance. Route-based lazy loading loads components when users navigate to routes:

```typescript
const routes = [
  { path: 'operations', loadChildren: () => import('./operations/operations.module') },
  { path: 'engineering', loadChildren: () => import('./engineering/engineering.module') }
];
```

Component-level lazy loading delays loading heavy components (network diagram viewer with large libraries) until they appear in the viewport.

## Real-Time Updates

### WebSocket Connections

WebSocket connections provide full-duplex communication channels for real-time data streaming. The client establishes WebSocket connections during initialization, maintains connections throughout the session with automatic reconnection on failures, and subscribes to relevant data streams based on user context.

WebSocket message handling integrates with state management. Incoming measurement updates dispatch actions that reducers apply to state, triggering component re-renders with new data. Message buffering prevents overload when updates arrive faster than the UI can render, combining multiple updates for the same measurement.

### Server-Sent Events (SSE)

Server-Sent Events provide simpler one-way streaming from server to client for notifications and updates. SSE uses standard HTTP, simplifying firewall traversal and load balancer configuration compared to WebSockets. The client subscribes to SSE endpoints delivering alarm notifications, system status updates, and progress events for long-running operations.

## Code Examples

### React Component with State Management

```typescript
import React from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { fetchEquipment, updateEquipment } from './equipmentSlice';

interface Equipment {
  id: string;
  name: string;
  status: string;
  voltage: number;
}

export const EquipmentList: React.FC = () => {
  const dispatch = useDispatch();
  const equipment = useSelector((state: RootState) => state.equipment.items);
  const loading = useSelector((state: RootState) => state.equipment.loading);

  React.useEffect(() => {
    dispatch(fetchEquipment());
  }, [dispatch]);

  const handleStatusChange = (id: string, newStatus: string) => {
    dispatch(updateEquipment({ id, status: newStatus }));
  };

  if (loading) {
    return <div>Loading equipment...</div>;
  }

  return (
    <div className="equipment-list">
      <h2>Equipment</h2>
      {equipment.map(item => (
        <div key={item.id} className="equipment-item">
          <h3>{item.name}</h3>
          <p>Status: {item.status}</p>
          <p>Voltage: {item.voltage} kV</p>
          <button onClick={() => handleStatusChange(item.id, 'OUT_OF_SERVICE')}>
            Take Out of Service
          </button>
        </div>
      ))}
    </div>
  );
};
```

### Redux State Management

```typescript
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import { equipmentApi } from '../api/equipmentApi';

interface EquipmentState {
  items: Equipment[];
  loading: boolean;
  error: string | null;
}

const initialState: EquipmentState = {
  items: [],
  loading: false,
  error: null
};

export const fetchEquipment = createAsyncThunk(
  'equipment/fetch',
  async () => {
    const response = await equipmentApi.getEquipment();
    return response.data;
  }
);

export const updateEquipment = createAsyncThunk(
  'equipment/update',
  async ({ id, status }: { id: string; status: string }) => {
    const response = await equipmentApi.updateEquipment(id, { status });
    return response.data;
  }
);

const equipmentSlice = createSlice({
  name: 'equipment',
  initialState,
  reducers: {
    equipmentUpdatedFromWebSocket: (state, action: PayloadAction<Equipment>) => {
      const index = state.items.findIndex(item => item.id === action.payload.id);
      if (index !== -1) {
        state.items[index] = action.payload;
      }
    }
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchEquipment.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchEquipment.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchEquipment.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to fetch equipment';
      });
  }
});

export const { equipmentUpdatedFromWebSocket } = equipmentSlice.actions;
export default equipmentSlice.reducer;
```

### WebSocket Integration

```typescript
class WebSocketService {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  connect(url: string, dispatch: Dispatch) {
    this.ws = new WebSocket(url);

    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;
      this.subscribe(['measurements', 'alarms']);
    };

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      this.handleMessage(message, dispatch);
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    this.ws.onclose = () => {
      console.log('WebSocket closed');
      this.attemptReconnect(url, dispatch);
    };
  }

  private handleMessage(message: any, dispatch: Dispatch) {
    switch (message.type) {
      case 'measurement':
        dispatch(measurementUpdated(message.data));
        break;
      case 'alarm':
        dispatch(alarmReceived(message.data));
        break;
      case 'equipment_status':
        dispatch(equipmentUpdatedFromWebSocket(message.data));
        break;
    }
  }

  private subscribe(topics: string[]) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({
        action: 'subscribe',
        topics: topics
      }));
    }
  }

  private attemptReconnect(url: string, dispatch: Dispatch) {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
      console.log(`Reconnecting in ${delay}ms... (attempt ${this.reconnectAttempts})`);
      setTimeout(() => this.connect(url, dispatch), delay);
    }
  }

  disconnect() {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }
}

export const webSocketService = new WebSocketService();
```

## Best Practices

- Use TypeScript for type safety, reducing runtime errors and improving developer experience with IDE support
- Implement centralized state management for complex applications to maintain predictable data flow
- Design components to be small, focused, and reusable with clear interfaces
- Separate business logic from presentation components to improve testability
- Implement proper error boundaries to gracefully handle component errors without crashing the entire application
- Use React.memo, useMemo, and useCallback to prevent unnecessary component re-renders
- Implement comprehensive logging and error tracking (Sentry, LogRocket) for production issue diagnosis
- Follow accessibility best practices (WCAG 2.1) with proper ARIA labels and keyboard navigation
- Configure source maps for production builds to enable error stack trace debugging while minifying code
- Implement automated testing at unit (component), integration, and end-to-end levels
