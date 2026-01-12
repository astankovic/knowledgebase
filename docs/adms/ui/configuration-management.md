---
layout: default
title: Configuration Management
parent: User Interface
grand_parent: ADMS Overview
nav_order: 4
---

# Configuration Management

## Overview

Configuration management enables users to customize ADMS interfaces to their specific roles, preferences, and workflows. Operators personalize dashboard layouts, display settings, alarm filters, and favorite equipment lists to optimize their productivity. Administrators configure system-wide settings including user permissions, notification rules, display defaults, and feature toggles. Effective configuration management balances flexibility (users control their experience) with consistency (maintain usability standards and operational safety).

Configuration data persists across sessions, synchronizing across multiple devices when users log in from different workstations. Cloud-based profile storage enables operators to access personalized configurations whether working from control center consoles, remote laptops, or mobile devices. Configuration changes take effect immediately without application restarts, using reactive state management to propagate updates throughout the UI.

Configuration interfaces follow progressive disclosure principles: common settings are prominently accessible while advanced options require navigating to dedicated configuration screens. Setting descriptions explain impacts clearly, validation prevents invalid configurations, and reset-to-defaults options provide safety nets when users make errors. Changes to safety-critical settings (alarm thresholds, control confirmations) require supervisory approval or multi-factor authentication.

## User Preferences

### Dashboard Layouts

Dashboard layout customization enables users to add, remove, resize, and reposition widgets displaying relevant information. Widget catalogs list available widget types (alarm list, measurement trending, equipment status summary, network diagram) with descriptions and preview images. Drag-and-drop interactions position widgets in responsive grid layouts that automatically reflow across different screen sizes.

Widget configuration dialogs customize widget content and appearance. An alarm list widget configures which severities to display, whether to show acknowledged alarms, how many alarms to list, and refresh intervals. Measurement trending widgets select which measurements to chart, time ranges, Y-axis scales, and line colors.

Multiple dashboard configurations save as named layouts: "Normal Operations", "Storm Restoration", "Load Peak Management". Users switch layouts via dropdown selectors, loading saved widget configurations instantly. Dashboard sharing enables users to export layouts as JSON files or share via URLs, facilitating standardization across operator teams.

### Display Settings

Display settings control visual appearance including themes, fonts, colors, and animation preferences. Light and dark themes toggle between bright backgrounds for well-lit environments and dark backgrounds for low-light control rooms. High-contrast modes enhance visibility for color-blind users or operators with visual impairments.

Font size controls scale text from 100% (default) to 150% (large text) without breaking layouts. Operators working on low-resolution displays or sitting farther from screens benefit from larger text. Animation preferences toggle transitions, reducing motion for users sensitive to animation or improving responsiveness on slower hardware.

Color-blind safe color schemes replace standard red/green status indicators with blue/orange or use shape indicators (circles/triangles/squares) in addition to colors. This ensures critical status information remains perceivable regardless of color vision deficiencies.

### Default Views

Default view configurations specify which pages load on application startup, which diagram regions display initially, and default time ranges for historical queries. Operators in different roles start with relevant contexts: system operators see enterprise-wide dashboards, substation operators see their assigned substations, engineering analysts see network analysis tools.

Default filters remember common query parameters: frequently searched equipment types, standard time ranges for trending, or favorite feeders for network diagrams. These defaults save repetitive configuration for routine operations while remaining overridable for ad-hoc needs.

## System Configuration

### Module Settings

Module configuration controls functionality available to users based on operational requirements and licensing. Feature toggles enable/disable entire modules (outage management, fault location, load management) or specific capabilities within modules. This simplifies interfaces for utilities not using certain features and prevents users from accessing unlicensed functionality.

Module settings configure operational parameters: alarm suppression durations, automatic logout timeouts for inactive sessions, measurement data refresh intervals, and calculation default parameters. Administrators balance system performance, network bandwidth, and operational needs when tuning these parameters.

Integration configurations specify external system connection parameters, data mapping rules, and synchronization schedules. Administrators configure GIS service URLs, SCADA protocol settings, and REST API authentication credentials through secure configuration interfaces that encrypt sensitive values.

### Role-Based UI

Role-based access control extends to UI configuration where different user roles see different interfaces tailored to their responsibilities. Operators see operational controls and monitoring views. Engineers see network modeling and analysis tools. Administrators see user management and system configuration interfaces.

Role configurations define which menu items appear, which widgets are available in catalogs, which actions are permitted on equipment, and which data is visible. Granular permissions control visibility of specific equipment types, geographic regions, or voltage levels based on user assignments.

Hierarchical roles inherit permissions from parent roles, simplifying administration. The Senior Operator role inherits all Operator permissions plus additional capabilities. Custom roles combine permissions from multiple base roles, accommodating utility-specific organizational structures.

### Localization

Localization support enables interface translation to multiple languages with region-specific formatting for dates, times, numbers, and currencies. Language selection appears in user preferences, applying immediately to all UI text except user-entered content (equipment names, alarm messages from field devices).

Translation files map message keys to localized strings with support for plural forms, gender agreement, and context-specific translations. Professional translations are provided for major languages; utilities can add custom translations for regional languages or utility-specific terminology.

Regional formatting applies locale-appropriate patterns: MM/DD/YYYY dates in the US, DD/MM/YYYY in Europe, 12-hour vs 24-hour time formats, decimal vs comma separators in numbers. Timezone configuration displays timestamps in user-preferred zones while storing all data in UTC.

## Configuration Storage

### Local Storage

Browser LocalStorage caches user preferences client-side for immediate loading without server round-trips. Cached data includes recently accessed equipment lists, last-used dashboard layout, current zoom/pan positions on diagrams, and form input history for autocomplete suggestions.

LocalStorage handles temporary data that enhances user experience but isn't critical to preserve: in-progress form drafts, expanded/collapsed panel states, and sort/filter selections on data tables. Cache expiration policies delete old data after inactivity periods, preventing unlimited growth.

### Server-Side Profiles

Server-side profile storage persists important user configurations centrally, enabling access from multiple devices and providing backup/restore capabilities. User profiles include dashboard layouts, display preferences, saved queries, and custom alarm filter configurations. Profiles sync to client on login and update server when users modify settings.

Profile versioning maintains configuration history, enabling rollback if users make undesirable changes. Administrators restore user profiles from backups or copy configurations between users for onboarding new operators with experienced operator settings.

Configuration APIs enable programmatic profile management. Bulk operations set default configurations for user groups, import/export profiles for disaster recovery, or migrate configurations between ADMS environments (test to production).

## Code Examples

### User Preferences Component

```typescript
interface UserPreferences {
  theme: 'light' | 'dark';
  fontSize: number;
  language: string;
  defaultDashboard: string;
  alarmSound: boolean;
}

const PreferencesDialog: React.FC = () => {
  const preferences = useSelector(selectUserPreferences);
  const dispatch = useDispatch();
  const [localPrefs, setLocalPrefs] = useState(preferences);

  const handleSave = () => {
    dispatch(updateUserPreferences(localPrefs));
    // Persist to server
    userApi.updatePreferences(localPrefs);
  };

  return (
    <Dialog open={true} onClose={() => {}}>
      <DialogTitle>User Preferences</DialogTitle>
      <DialogContent>
        <FormControl fullWidth margin="normal">
          <InputLabel>Theme</InputLabel>
          <Select
            value={localPrefs.theme}
            onChange={(e) => setLocalPrefs({ ...localPrefs, theme: e.target.value as any })}
          >
            <MenuItem value="light">Light</MenuItem>
            <MenuItem value="dark">Dark</MenuItem>
          </Select>
        </FormControl>

        <FormControl fullWidth margin="normal">
          <InputLabel>Font Size</InputLabel>
          <Select
            value={localPrefs.fontSize}
            onChange={(e) => setLocalPrefs({ ...localPrefs, fontSize: Number(e.target.value) })}
          >
            <MenuItem value={100}>Normal (100%)</MenuItem>
            <MenuItem value={125}>Large (125%)</MenuItem>
            <MenuItem value={150}>Extra Large (150%)</MenuItem>
          </Select>
        </FormControl>

        <FormControl fullWidth margin="normal">
          <InputLabel>Language</InputLabel>
          <Select
            value={localPrefs.language}
            onChange={(e) => setLocalPrefs({ ...localPrefs, language: e.target.value })}
          >
            <MenuItem value="en">English</MenuItem>
            <MenuItem value="es">Español</MenuItem>
            <MenuItem value="fr">Français</MenuItem>
          </Select>
        </FormControl>

        <FormControlLabel
          control={
            <Checkbox
              checked={localPrefs.alarmSound}
              onChange={(e) => setLocalPrefs({ ...localPrefs, alarmSound: e.target.checked })}
            />
          }
          label="Enable alarm sounds"
        />
      </DialogContent>
      <DialogActions>
        <Button onClick={handleSave} color="primary">Save</Button>
        <Button onClick={() => setLocalPrefs(preferences)}>Cancel</Button>
      </DialogActions>
    </Dialog>
  );
};
```

### Dashboard Configuration Persistence

```typescript
interface DashboardConfig {
  id: string;
  name: string;
  widgets: Array<{
    id: string;
    type: string;
    position: { x: number; y: number };
    size: { width: number; height: number };
    config: any;
  }>;
}

class DashboardConfigService {
  async saveDashboard(userId: string, dashboard: DashboardConfig): Promise<void> {
    // Save to server
    await api.post(`/users/${userId}/dashboards`, dashboard);

    // Cache locally
    localStorage.setItem(
      `dashboard_${dashboard.id}`,
      JSON.stringify(dashboard)
    );
  }

  async loadDashboard(userId: string, dashboardId: string): Promise<DashboardConfig> {
    // Try local cache first
    const cached = localStorage.getItem(`dashboard_${dashboardId}`);
    if (cached) {
      return JSON.parse(cached);
    }

    // Fetch from server
    const response = await api.get(`/users/${userId}/dashboards/${dashboardId}`);
    const dashboard = response.data;

    // Cache for next time
    localStorage.setItem(`dashboard_${dashboardId}`, JSON.stringify(dashboard));

    return dashboard;
  }

  async listDashboards(userId: string): Promise<DashboardConfig[]> {
    const response = await api.get(`/users/${userId}/dashboards`);
    return response.data;
  }

  async deleteDashboard(userId: string, dashboardId: string): Promise<void> {
    await api.delete(`/users/${userId}/dashboards/${dashboardId}`);
    localStorage.removeItem(`dashboard_${dashboardId}`);
  }
}
```

### Localization Implementation

```typescript
interface TranslationMessages {
  [key: string]: string;
}

class LocalizationService {
  private messages: Record<string, TranslationMessages> = {};
  private currentLocale: string = 'en';

  async loadLocale(locale: string): Promise<void> {
    if (!this.messages[locale]) {
      const response = await fetch(`/locales/${locale}.json`);
      this.messages[locale] = await response.json();
    }
    this.currentLocale = locale;
  }

  translate(key: string, params?: Record<string, any>): string {
    let message = this.messages[this.currentLocale]?.[key] || key;

    // Replace parameters
    if (params) {
      Object.entries(params).forEach(([param, value]) => {
        message = message.replace(`{${param}}`, String(value));
      });
    }

    return message;
  }

  formatDate(date: Date): string {
    return new Intl.DateTimeFormat(this.currentLocale).format(date);
  }

  formatNumber(value: number, decimals: number = 2): string {
    return new Intl.NumberFormat(this.currentLocale, {
      minimumFractionDigits: decimals,
      maximumFractionDigits: decimals
    }).format(value);
  }
}

// Usage in component
const AlarmMessage: React.FC<{ count: number }> = ({ count }) => {
  const { t } = useTranslation();

  return (
    <div>
      {t('alarms.active_count', { count })}
      {/* Renders "5 active alarms" in English, "5 alarmes actives" in French */}
    </div>
  );
};
```

## Best Practices

- Store critical configuration (dashboard layouts, display preferences) server-side for multi-device access and backup
- Cache frequently accessed configuration client-side to improve performance and enable offline functionality
- Validate configuration changes before applying to prevent invalid states that could impact operations
- Provide configuration import/export capabilities for backup, sharing, and migration between environments
- Implement configuration versioning to enable rollback and audit trails of configuration changes
- Use secure storage and encryption for sensitive configuration values like API credentials and passwords
- Apply configuration changes immediately without requiring application restarts for better user experience
- Provide sensible default configurations that work out-of-the-box while enabling customization
- Document configuration options thoroughly with descriptions, valid ranges, and impact explanations
- Test configuration interfaces with actual users to ensure intuitiveness and prevent configuration errors
