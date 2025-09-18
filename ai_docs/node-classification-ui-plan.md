# Node Classification UI Implementation Plan

**Date**: September 18, 2024  
**Branch**: `node-classification-ui`  
**Based on**: Denver Mesh Network Node Categorization Analysis  
**Status**: Planning Phase

## Overview

This document outlines the implementation plan for adding node classification features to the PotatoMesh web UI. The goal is to replace the current colored dots on the map with meaningful icons/glyphs that represent different node types, and provide users with a toggle between the original view and the new classified view.

## Current State Analysis

### Existing Map Implementation
- **Current markers**: Colored circle markers based on node role (CLIENT, ROUTER, etc.)
- **Color scheme**: Role-based colors defined in `roleColors` object
- **Legend**: Shows role colors and meanings
- **Data source**: Node data from Meshtastic ingestor via `/nodes` API endpoint

### Current Data Collection
The ingestor script (`data/mesh.py`) currently collects:
- Node ID, user info, hardware type, role
- Position data (latitude/longitude)
- Battery status, last heard timestamps
- All data is serialized via `_node_to_dict()` function

## Implementation Plan

### Phase 1: Frontend-Only Implementation

**No backend changes needed!** The frontend can classify nodes in real-time using existing data.

#### 1.1 Frontend Classification Logic
**File**: `web/views/index.erb` (JavaScript section)

**Classification function**:
```javascript
function classifyNode(node) {
  // Priority 1: Hardware-based classification
  if (node.hw_model === 'T_DECK') {
    return 'tdeck';
  } else if (node.hw_model === 'T_ECHO') {
    return 'techo';
  }
  
  // Priority 2: Role-based classification
  if (node.role === 'ROUTER' || node.role === 'ROUTER_LATE') {
    return 'bbs_server';
  }
  
  // Priority 3: Keyword-based classification
  const nameText = `${node.user || ''} ${node.aka || ''}`.toLowerCase();
  
  if (['solar', 'sun', '☀'].some(kw => nameText.includes(kw))) {
    return 'solar';
  } else if (['mobile', 'car', 'vehicle'].some(kw => nameText.includes(kw))) {
    return 'mobile';
  } else if (['base', 'home', 'qth', 'fixed'].some(kw => nameText.includes(kw))) {
    return 'base_station';
  } else if (['weather', 'temp', 'sensor'].some(kw => nameText.includes(kw))) {
    return 'weather';
  } else if (['bbs', 'station', 'repeater', 'rptr'].some(kw => nameText.includes(kw))) {
    return 'bbs_server';
  }
  
  return 'other';
}

function getNodeIcon(category) {
  const iconMap = {
    'bbs_server': '📡',
    'weather': '🌡️',
    'mobile': '🚗',
    'base_station': '🏠',
    'solar': '☀️',
    'tdeck': '⌨️',
    'techo': '📱',
    'other': '📡'
  };
  return iconMap[category] || '📡';
}

function getHardwareIcon(hwModel) {
  const hardwareIconMap = {
    // T-Deck devices
    'T_DECK': '⌨️',
    
    // T-Echo devices  
    'T_ECHO': '📱',
    
    // Heltec devices
    'HELTEC_V3': '📻',
    'HELTEC_V4': '📻',
    'HELTEC_V5': '📻',
    
    // RAK devices
    'RAK4631': '📡',
    'RAK11200': '📡',
    'RAK11300': '📡',
    'RAK4631_5005': '📡',
    
    // Station devices
    'STATION_G1': '🏗️',
    'STATION_G2': '🏗️',
    
    // Tracker devices
    'TRACKER_T1000_E': '🌡️',
    'TRACKER_T1000': '📍',
    
    // Generic categories for unknown models
    'HELTEC': '📻',
    'RAK': '📡',
    'STATION': '🏗️',
    'TRACKER': '📍'
  };
  
  // Try exact match first
  if (hardwareIconMap[hwModel]) {
    return hardwareIconMap[hwModel];
  }
  
  // Try partial matches for unknown variants
  const hwUpper = hwModel.toUpperCase();
  if (hwUpper.includes('HELTEC')) return '📻';
  if (hwUpper.includes('RAK')) return '📡';
  if (hwUpper.includes('STATION')) return '🏗️';
  if (hwUpper.includes('TRACKER')) return '📍';
  if (hwUpper.includes('T_DECK')) return '⌨️';
  if (hwUpper.includes('T_ECHO')) return '📱';
  
  // Default fallback
  return '📡';
}
```

### Phase 2: Frontend UI Updates

#### 2.1 Map View Toggle
**File**: `web/views/index.erb`

**New UI elements**:
- Toggle button/selector for map view modes
- Three view options: Original, Classified, and Hardware
- Preserve existing functionality

**Implementation**:
```html
<div class="map-controls">
  <label>Map View:</label>
  <select id="mapViewMode">
    <option value="original">Original (Role-based)</option>
    <option value="classified">Classified (Node Types)</option>
    <option value="hardware">Hardware (Device Models)</option>
  </select>
</div>
```

#### 2.2 Icon-based Markers
**Replace circle markers with icon markers based on view mode**:

```javascript
// Icon marker creation
function createIconMarker(node, viewMode) {
  let iconHtml, iconClass;
  
  switch(viewMode) {
    case 'original':
      // Use existing circle markers with role colors
      return L.circleMarker([node.latitude, node.longitude], {
        radius: 9,
        color: '#000',
        weight: 1,
        fillColor: roleColors[node.role] || '#3388ff',
        fillOpacity: 0.7,
        opacity: 0.7
      });
      
    case 'classified':
      const category = classifyNode(node);
      iconHtml = getNodeIcon(category);
      iconClass = `node-icon classified ${category}`;
      break;
      
    case 'hardware':
      iconHtml = getHardwareIcon(node.hw_model);
      iconClass = `node-icon hardware ${node.hw_model}`;
      break;
  }
  
  const icon = L.divIcon({
    html: `<div class="${iconClass}">${iconHtml}</div>`,
    className: 'custom-div-icon',
    iconSize: [24, 24],
    iconAnchor: [12, 12]
  });
  
  return L.marker([node.latitude, node.longitude], { icon });
}
```

#### 2.3 Enhanced Tooltips
**Adjective categories in tooltips**:

```javascript
function createNodeTooltip(node) {
  const baseInfo = [
    `Node: ${node.user || 'Unknown'}`,
    `Short: ${node.aka || 'N/A'}`,
    `Category: ${getCategoryDisplayName(node.category)}`,
    `Hardware: ${node.hw_model || 'Unknown'}`
  ];
  
  // Add adjective categories
  const adjectives = [];
  if (isSolarNode(node)) adjectives.push('☀️ Solar-powered');
  if (isMobileNode(node)) adjectives.push('🚗 Mobile');
  if (isWeatherNode(node)) adjectives.push('🌡️ Weather station');
  
  if (adjectives.length > 0) {
    baseInfo.push(`Features: ${adjectives.join(', ')}`);
  }
  
  return baseInfo.join('<br/>');
}
```

#### 2.4 Updated Legend
**Dynamic legend based on view mode**:

```javascript
function updateLegend(viewMode) {
  const legendDiv = document.querySelector('.legend');
  
  switch(viewMode) {
    case 'original':
      // Show role-based colors
      legendDiv.innerHTML = Object.entries(roleColors)
        .map(([role, color]) => `<div><span style="background:${color}"></span>${role}</div>`)
        .join('');
      break;
        
    case 'classified':
      // Show category-based icons
      const categoryDisplayNames = {
        'bbs_server': 'BBS Server',
        'weather': 'Weather Station',
        'mobile': 'Mobile Node',
        'base_station': 'Base Station',
        'solar': 'Solar Node',
        'tdeck': 'T-Deck',
        'techo': 'T-Echo',
        'other': 'Other'
      };
      legendDiv.innerHTML = Object.entries(categoryDisplayNames)
        .map(([category, name]) => `<div><span class="legend-icon">${getNodeIcon(category)}</span>${name}</div>`)
        .join('');
      break;
        
    case 'hardware':
      // Show hardware-based icons
      const hardwareDisplayNames = {
        'T_DECK': 'T-Deck',
        'T_ECHO': 'T-Echo',
        'HELTEC_V3': 'Heltec V3',
        'HELTEC_V4': 'Heltec V4',
        'HELTEC_V5': 'Heltec V5',
        'RAK4631': 'RAK4631',
        'RAK11200': 'RAK11200',
        'RAK11300': 'RAK11300',
        'STATION_G1': 'Station G1',
        'STATION_G2': 'Station G2',
        'TRACKER_T1000': 'Tracker T1000',
        'TRACKER_T1000_E': 'Tracker T1000-E'
      };
      legendDiv.innerHTML = Object.entries(hardwareDisplayNames)
        .map(([hw, name]) => `<div><span class="legend-icon">${getHardwareIcon(hw)}</span>${name}</div>`)
        .join('');
      break;
  }
}
```

### Phase 3: Styling and UX

#### 3.1 CSS Updates
**File**: `web/views/index.erb` (style section)

```css
/* Node icon styling */
.node-icon {
  font-size: 20px;
  text-align: center;
  line-height: 24px;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 50%;
  border: 2px solid #333;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.custom-div-icon {
  background: transparent;
  border: none;
}

/* Legend icon styling */
.legend-icon {
  display: inline-block;
  width: 16px;
  text-align: center;
  margin-right: 6px;
}

/* Map controls */
.map-controls {
  position: absolute;
  top: 10px;
  right: 10px;
  background: rgba(255, 255, 255, 0.9);
  padding: 8px;
  border-radius: 4px;
  border: 1px solid #ccc;
  z-index: 1000;
}

/* Dark mode support */
body.dark .node-icon {
  background: rgba(0, 0, 0, 0.9);
  border-color: #666;
}

body.dark .map-controls {
  background: rgba(0, 0, 0, 0.9);
  border-color: #444;
  color: #eee;
}
```

#### 3.2 Responsive Design
- Ensure icons are visible on mobile devices
- Maintain touch-friendly marker sizes
- Preserve existing mobile layout

### Phase 4: Testing and Validation

#### 4.1 Test Cases
- [ ] Toggle between original and classified views
- [ ] Icon rendering on different screen sizes
- [ ] Tooltip accuracy for adjective categories
- [ ] Legend updates correctly
- [ ] Dark mode compatibility
- [ ] Performance with large node counts

#### 4.2 Data Validation
- [ ] Classification accuracy against known nodes
- [ ] Edge cases (nodes with multiple characteristics)
- [ ] Fallback behavior for unclassified nodes

## Map View Modes

### 1. Original View (Role-based)
- **Colored circle markers** based on node role
- **Existing functionality** preserved
- **Role colors**: CLIENT, ROUTER, etc.

### 2. Classified View (Node Types)
Based on the Denver Mesh analysis:

| Category | Icon | Description | Examples |
|----------|------|-------------|----------|
| `bbs_server` | 📡 | BBS/Repeater stations | "LRA rptr. W0XYZ", "POSTAL REPEATER" |
| `weather` | 🌡️ | Weather monitoring | "WnQ Temp" |
| `mobile` | 🚗 | Mobile/vehicle nodes | "KF0KIT/Patzy - Mobile Node" |
| `base_station` | 🏠 | Fixed home/base stations | "GZ Home Base", "Justin Base 9bb6" |
| `solar` | ☀️ | Solar-powered nodes | "W3OO Solar Heltec V4" |
| `tdeck` | ⌨️ | T-Deck devices | Hardware: T_DECK |
| `techo` | 📱 | T-Echo devices | Hardware: T_ECHO |
| `other` | 📡 | Unclassified nodes | Default category |

### 3. Hardware View (Device Models)
Device-specific icons for different hardware models:

| Hardware Model | Icon | Description |
|----------------|------|-------------|
| `T_DECK` | ⌨️ | T-Deck keyboard device |
| `T_ECHO` | 📱 | T-Echo handheld device |
| `HELTEC_V3` | 📻 | Heltec V3 LoRa module |
| `HELTEC_V4` | 📻 | Heltec V4 LoRa module |
| `HELTEC_V5` | 📻 | Heltec V5 LoRa module |
| `RAK4631` | 📡 | RAK4631 LoRa module |
| `RAK11200` | 📡 | RAK11200 LoRa module |
| `RAK11300` | 📡 | RAK11300 LoRa module |
| `STATION_G1` | 🏗️ | Station G1 base station |
| `STATION_G2` | 🏗️ | Station G2 base station |
| `TRACKER_T1000` | 📍 | Tracker T1000 GPS device |
| `TRACKER_T1000_E` | 🌡️ | Tracker T1000-E environmental |
| Unknown/Other | 📡 | Generic radio icon |

## Adjective Categories (Tooltip Features)

These are additional characteristics that can be combined with primary categories:

- **☀️ Solar-powered**: Nodes with "solar", "sun", or "☀" in name
- **🚗 Mobile**: Nodes with "mobile", "car", "vehicle" in name  
- **🌡️ Weather**: Nodes with "weather", "temp", "sensor" in name
- **🔋 Battery**: Nodes with low battery status
- **📶 Strong Signal**: Nodes with high signal strength

## Implementation Approach

The classification will be based on a combination of:

1. **Hardware Type** (most deterministic) - T_DECK, T_ECHO, etc.
2. **Role Field** (deterministic) - ROUTER, CLIENT, etc.
3. **Keyword Analysis** (heuristic) - parsing user names and AKA fields

The keyword approach is what I was planning, but we should first examine the actual node list output to see what deterministic metadata is available beyond just hardware and role fields.

### Data Structure Analysis Needed

Before implementing, we should examine the actual node data structure from the Meshtastic CLI to identify:

- **Available fields** in the node objects
- **Deterministic indicators** beyond hardware/role
- **Telemetry data** that might indicate node type
- **Configuration fields** that could provide classification hints

This analysis will help determine if we can rely more on deterministic data rather than keyword parsing.

## Success Metrics

- [ ] Users can toggle between original and classified views
- [ ] Icons accurately represent node types
- [ ] Tooltips provide meaningful additional information
- [ ] Performance remains acceptable with large node counts
- [ ] Dark mode and mobile compatibility maintained
- [ ] Classification accuracy > 90% for known node types

## Future Enhancements

- **Machine Learning**: Improve classification with usage patterns
- **User Feedback**: Allow users to correct classifications
- **Custom Icons**: Support for custom icon sets
- **Advanced Filtering**: Filter map by node categories
- **Statistics Dashboard**: Show category distribution
- **Export Features**: Export classified node data

## Technical Considerations

### Performance
- Classification should be done server-side to avoid client-side processing
- Cache classification results to avoid repeated computation
- Consider database indexing for category-based queries

### Accessibility
- Ensure icons have proper alt text for screen readers
- Maintain keyboard navigation for map controls
- Provide text alternatives for icon-based information

### Browser Compatibility
- Test icon rendering across different browsers
- Fallback to text-based markers if icons fail
- Ensure Leaflet compatibility with custom markers

## Conclusion

This implementation plan provides a comprehensive approach to adding node classification features to the PotatoMesh web UI. The phased approach ensures minimal disruption to existing functionality while providing users with valuable new insights into their mesh network topology.

The key innovation is the dual-view system that preserves the familiar role-based view while introducing the more intuitive icon-based classification system. The adjective categories in tooltips provide additional context without cluttering the main interface.
