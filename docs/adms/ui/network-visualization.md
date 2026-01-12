---
layout: default
title: Network Visualization
parent: User Interface
grand_parent: ADMS Overview
nav_order: 2
---

# Network Visualization

## Overview

Network visualization renders interactive electrical network diagrams displaying substations, feeders, transformers, switches, and other equipment with real-time status indications, measurement values, and operational states. These diagrams serve as the primary operator interface for situational awareness, enabling operators to understand network topology, identify equipment conditions, monitor power flows, and execute control operations through intuitive graphical interactions.

The visualization challenge lies in rendering large-scale networks (thousands of equipment objects) with smooth performance, maintaining visual clarity at different zoom levels, updating displays in real-time as conditions change, and providing rich interactions without overwhelming users. Advanced rendering technologies including SVG for scalable vector graphics, HTML5 Canvas for high-performance pixel manipulation, and WebGL for hardware-accelerated 3D rendering enable meeting these requirements.

Diagram layouts follow utility conventions with geographic accuracy (equipment positioned at actual locations) or schematic layouts (equipment arranged for visual clarity prioritizing electrical connectivity over geography). Symbology standards (IEEE, IEC) ensure equipment symbols are immediately recognizable to trained operators. Color coding indicates voltage levels, energization status, and alarm conditions following industry conventions.

## Rendering Technologies

### SVG-Based Rendering

Scalable Vector Graphics (SVG) provides resolution-independent rendering where graphics scale without quality loss. SVG is DOM-based with each equipment symbol represented as an SVG element, enabling CSS styling, event handling, and animations. This approach simplifies interactions: clicking equipment triggers event handlers, hovering displays tooltips, and dragging equipment moves symbols interactively.

SVG rendering excels for moderate-sized diagrams (hundreds to low thousands of elements) where full interactivity is required. Equipment symbols are SVG groups containing paths, circles, and text elements. Real-time updates modify element attributes: changing breaker color from green (closed) to red (open), updating measurement text values, or adding alarm indicator overlays.

Performance degrades with large element counts as browser DOM manipulation becomes expensive. Techniques to optimize include: using symbol definitions with `<use>` elements to avoid duplicating complex symbols, removing non-visible elements from the DOM, and throttling updates to avoid excessive repaints.

### Canvas-Based Rendering

HTML5 Canvas provides pixel-based rendering through JavaScript drawing APIs. Canvas excels at rendering large numbers of objects (tens of thousands) with consistent performance as graphics are drawn directly to pixel buffers without DOM overhead. The canvas acts as a single bitmap; individual equipment symbols don't exist as discrete objects in the DOM.

Canvas rendering requires manual hit testing for interactions. Clicking the canvas triggers coordinate-to-equipment mapping through spatial indexes (R-trees, quadtrees) identifying which equipment falls under the click point. Rendering loops draw visible equipment based on viewport and zoom level, implementing frustum culling to skip off-screen objects.

Canvas is ideal for overview diagrams displaying entire distribution networks where equipment density is high. The trade-off is implementation complexity: developers manually handle drawing, interactions, and updates rather than leveraging browser DOM capabilities.

### WebGL Rendering

WebGL provides hardware-accelerated 3D graphics using GPU rendering for maximum performance. WebGL handles extremely large datasets (hundreds of thousands of objects) with real-time updates. Graphics libraries (Three.js, PixiJS) abstract low-level WebGL programming while maintaining performance benefits.

WebGL rendering represents equipment as textured sprites or 3D models rendered through shader programs executing on GPUs. Batch rendering draws multiple equipment symbols in single draw calls. Instanced rendering reuses geometry for repeated symbols (all breakers use the same base geometry with different transforms).

WebGL suits specialized visualizations like 3D substation models, heat maps overlaying distribution networks with color-coded data, or particle effects showing power flow animations. However, WebGL adds complexity and debugging difficulty compared to SVG/Canvas approaches.

## Network Diagram Features

### Symbol Library

Equipment symbols follow standardized conventions with distinct shapes for each equipment type: transformers as circles with winding indicators, breakers as rectangles with contact separation, switches as angled lines, and buses as thick horizontal/vertical lines. Symbol libraries include variants for single-phase, two-phase, and three-phase equipment, different voltage classes, and various equipment subtypes.

Symbols scale appropriately with zoom level, maintaining readability. At wide zoom showing hundreds of substations, symbols simplify to basic shapes. Zooming in reveals details like transformer winding configurations, relay flags, and measurement values. Dynamic level-of-detail algorithms select symbol complexity based on zoom.

### Coloring Schemes

Color coding communicates equipment status at-a-glance. Standard schemes include:

- **Voltage Level**: Different colors distinguish voltage classes (red for transmission, blue for subtransmission, green for primary distribution, gray for secondary)
- **Energization Status**: Equipment colors indicate energized (voltage present) vs de-energized states
- **Alarm Status**: Flashing red borders indicate active alarms, yellow indicates warnings
- **Selection**: Selected equipment highlights with bright outline or color change

User-configured color schemes accommodate color-blindness and personal preferences while maintaining operational safety by preserving critical status indications.

### Dynamic Updates

Real-time measurement and status updates modify diagram appearance immediately. Breaker status changes animate the symbol from closed (continuous line) to open (line with gap). Current flow indicators show animated arrows along conductors with speed proportional to load. Voltage violations color equipment red for over-voltage, yellow for under-voltage.

Update throttling prevents excessive rendering when data arrives faster than display refresh rates. Batch updates collect multiple changes and apply them in single render passes. Dirty tracking identifies changed elements, rendering only affected diagram regions rather than entire diagrams.

## Interaction Patterns

### Pan and Zoom

Mouse drag pans the diagram, shifting the viewport across the network. Mouse wheel or pinch gestures zoom in/out, maintaining smooth 60 FPS rendering during zoom animations. Double-click zooms to fit selected equipment, automatically calculating bounds and applying appropriate zoom level.

Pan and zoom state persists across sessions, remembering user's last viewport when returning to diagrams. Minimap overviews show current viewport region within the overall network, enabling quick navigation to distant areas.

### Selection and Highlighting

Clicking equipment selects it, highlighting with colored outline and displaying details panel with attributes, measurements, and alarms. Shift-click enables multi-select. Marquee selection (drag rectangle) selects all enclosed equipment. Keyboard shortcuts (Ctrl+A) select all visible equipment.

Hover interactions display tooltips with equipment name, type, and current measurements without requiring clicks. Hover highlighting uses subtle color changes or glow effects distinct from selection highlighting. Hover state clears automatically when mouse leaves, avoiding persistent visual clutter.

### Context Menus

Right-clicking equipment opens context menus with available operations: View Details, Open Control Dialog, Trace Connectivity, View Alarms, View Measurements. Menu contents adapt based on equipment type and user permissions. Inaccessible operations appear grayed-out with explanatory tooltips.

Context menus support keyboard navigation with arrow keys and Enter to select. Menus close when clicking outside, pressing Escape, or selecting an action. Recent actions appear at menu tops for quick access.

## Code Examples

### SVG Rendering Component

```typescript
interface Equipment {
  id: string;
  type: string;
  x: number;
  y: number;
  status: string;
  voltage: number;
}

const NetworkDiagram: React.FC<{ equipment: Equipment[] }> = ({ equipment }) => {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [transform, setTransform] = useState({ x: 0, y: 0, scale: 1 });

  const handleEquipmentClick = (id: string) => {
    setSelectedId(id);
  };

  const getSymbolColor = (eq: Equipment) => {
    if (eq.status === 'ALARM') return 'red';
    if (eq.status === 'ENERGIZED') return 'green';
    return 'gray';
  };

  return (
    <svg width="100%" height="100%" viewBox="0 0 1000 1000">
      <g transform={`translate(${transform.x},${transform.y}) scale(${transform.scale})`}>
        {equipment.map(eq => (
          <g
            key={eq.id}
            transform={`translate(${eq.x},${eq.y})`}
            onClick={() => handleEquipmentClick(eq.id)}
            style={{ cursor: 'pointer' }}
          >
            {/* Render equipment symbol based on type */}
            {eq.type === 'BREAKER' && (
              <rect
                x="-10"
                y="-20"
                width="20"
                height="40"
                fill={getSymbolColor(eq)}
                stroke={selectedId === eq.id ? 'blue' : 'black'}
                strokeWidth={selectedId === eq.id ? 3 : 1}
              />
            )}
            {eq.type === 'TRANSFORMER' && (
              <circle
                r="15"
                fill={getSymbolColor(eq)}
                stroke={selectedId === eq.id ? 'blue' : 'black'}
                strokeWidth={selectedId === eq.id ? 3 : 1}
              />
            )}
            <text x="20" y="5" fontSize="12" fill="black">
              {eq.voltage.toFixed(1)} kV
            </text>
          </g>
        ))}
      </g>
    </svg>
  );
};
```

### Canvas Rendering

```javascript
class CanvasNetworkRenderer {
  constructor(canvas, equipment) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.equipment = equipment;
    this.viewport = { x: 0, y: 0, scale: 1 };
  }

  render() {
    // Clear canvas
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // Apply viewport transform
    this.ctx.save();
    this.ctx.translate(this.viewport.x, this.viewport.y);
    this.ctx.scale(this.viewport.scale, this.viewport.scale);

    // Render equipment
    for (const eq of this.equipment) {
      if (this.isVisible(eq)) {
        this.drawEquipment(eq);
      }
    }

    this.ctx.restore();
  }

  drawEquipment(eq) {
    this.ctx.save();
    this.ctx.translate(eq.x, eq.y);

    // Set color based on status
    this.ctx.fillStyle = this.getColor(eq.status);
    this.ctx.strokeStyle = eq.selected ? 'blue' : 'black';
    this.ctx.lineWidth = eq.selected ? 3 : 1;

    // Draw symbol based on type
    if (eq.type === 'BREAKER') {
      this.ctx.fillRect(-10, -20, 20, 40);
      this.ctx.strokeRect(-10, -20, 20, 40);
    } else if (eq.type === 'TRANSFORMER') {
      this.ctx.beginPath();
      this.ctx.arc(0, 0, 15, 0, Math.PI * 2);
      this.ctx.fill();
      this.ctx.stroke();
    }

    // Draw label
    this.ctx.fillStyle = 'black';
    this.ctx.font = '12px Arial';
    this.ctx.fillText(`${eq.voltage.toFixed(1)} kV`, 20, 5);

    this.ctx.restore();
  }

  isVisible(eq) {
    // Simple frustum culling
    const screenX = eq.x * this.viewport.scale + this.viewport.x;
    const screenY = eq.y * this.viewport.scale + this.viewport.y;
    return screenX >= -50 && screenX <= this.canvas.width + 50 &&
           screenY >= -50 && screenY <= this.canvas.height + 50;
  }

  getColor(status) {
    switch (status) {
      case 'ALARM': return 'red';
      case 'ENERGIZED': return 'green';
      default: return 'gray';
    }
  }

  hitTest(x, y) {
    // Convert screen coordinates to world coordinates
    const worldX = (x - this.viewport.x) / this.viewport.scale;
    const worldY = (y - this.viewport.y) / this.viewport.scale;

    // Find equipment at coordinates
    for (const eq of this.equipment) {
      const dx = worldX - eq.x;
      const dy = worldY - eq.y;
      const distance = Math.sqrt(dx * dx + dy * dy);

      if (distance < 20) { // Hit radius
        return eq;
      }
    }
    return null;
  }
}
```

## Best Practices

- Choose rendering technology based on diagram complexity: SVG for small-medium diagrams, Canvas for large diagrams, WebGL for extreme scale
- Implement level-of-detail rendering where symbol complexity adapts to zoom level
- Use spatial indexing (R-tree, quadtree) for efficient hit testing and viewport culling in large diagrams
- Throttle real-time updates to display refresh rate (60 FPS) to prevent excessive rendering
- Provide visual feedback for all interactions with hover effects, selection highlights, and click animations
- Follow industry symbol and color conventions to ensure operator familiarity and reduce training requirements
- Implement smooth pan and zoom animations using requestAnimationFrame for 60 FPS performance
- Enable keyboard shortcuts for common operations (arrow keys for panning, +/- for zoom, Esc to clear selection)
- Test rendering performance with realistic dataset sizes including worst-case scenarios
- Provide multiple diagram views (geographic layout, schematic layout, list view) to accommodate different operational needs
