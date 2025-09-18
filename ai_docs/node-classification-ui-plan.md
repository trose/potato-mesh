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

### Phase 1: Data Layer Enhancements

#### 1.1 Ingestor Script Updates
**File**: `data/mesh.py`

**Changes needed**:
- ✅ **No additional data collection required** - all necessary fields are already collected
- The ingestor already captures:
  - `hardware` field (T_DECK, T_ECHO, etc.)
  - `role` field (ROUTER, CLIENT, etc.)
  - `user` and `aka` fields for keyword analysis
  - All other required data for classification

**Classification Logic**:
```python
def classify_node(node_data):
    """Classify node based on hardware, role, and keywords."""
    # Priority 1: Hardware-based classification
    if node_data.get('hardware') == 'T_DECK':
        return 'tdeck'
    elif node_data.get('hardware') == 'T_ECHO':
        return 'techo'
    
    # Priority 2: Role-based classification
    if node_data.get('role') in ['ROUTER', 'ROUTER_LATE']:
        return 'bbs_server'
    
    # Priority 3: Keyword-based classification
    name_text = f"{node_data.get('user', '')} {node_data.get('aka', '')}".lower()
    
    if any(kw in name_text for kw in ['solar', 'sun', '☀']):
        return 'solar'
    elif any(kw in name_text for kw in ['mobile', 'car', 'vehicle']):
        return 'mobile'
    elif any(kw in name_text for kw in ['base', 'home', 'qth', 'fixed']):
        return 'base_station'
    elif any(kw in name_text for kw in ['weather', 'temp', 'sensor']):
        return 'weather'
    elif any(kw in name_text for kw in ['bbs', 'station', 'repeater', 'rptr']):
        return 'bbs_server'
    
    return 'other'
```

#### 1.2 Database Schema Updates
**File**: `data/messages.sql` (if needed)

**Optional enhancement**:
```sql
-- Add classification fields to nodes table
ALTER TABLE nodes ADD COLUMN category VARCHAR(20);
ALTER TABLE nodes ADD COLUMN classification_confidence DECIMAL(3,2);
ALTER TABLE nodes ADD COLUMN classification_method VARCHAR(20);
```

### Phase 2: Backend API Updates

#### 2.1 Node Classification Service
**File**: `web/app.rb`

**New endpoint**: `/nodes/classified`
- Returns nodes with classification data
- Applies classification algorithm to existing node data
- Maintains backward compatibility with existing `/nodes` endpoint

**Implementation**:
```ruby
get '/nodes/classified' do
  nodes = get_nodes_from_db
  classified_nodes = nodes.map do |node|
    node.merge(
      'category' => classify_node(node),
      'icon' => get_node_icon(classify_node(node)),
      'display_name' => get_display_name(node)
    )
  end
  classified_nodes.to_json
end
```

#### 2.2 Classification Helper Functions
```ruby
def classify_node(node_data)
  # Ruby implementation of classification logic
  # (mirrors Python version above)
end

def get_node_icon(category)
  icon_map = {
    'bbs_server' => '📡',
    'weather' => '🌡️',
    'mobile' => '🚗',
    'base_station' => '🏠',
    'solar' => '☀️',
    'tdeck' => '⌨️',
    'techo' => '📱',
    'other' => '📡'
  }
  icon_map[category] || '📡'
end
```

### Phase 3: Frontend UI Updates

#### 3.1 Map View Toggle
**File**: `web/views/index.erb`

**New UI elements**:
- Toggle button/selector for map view modes
- "Original" vs "Classified" view options
- Preserve existing functionality

**Implementation**:
```html
<div class="map-controls">
  <label>Map View:</label>
  <select id="mapViewMode">
    <option value="original">Original (Role-based)</option>
    <option value="classified">Classified (Node Types)</option>
  </select>
</div>
```

#### 3.2 Icon-based Markers
**Replace circle markers with icon markers**:

```javascript
// Icon marker creation
function createIconMarker(node, category) {
  const icon = L.divIcon({
    html: `<div class="node-icon ${category}">${getNodeIcon(category)}</div>`,
    className: 'custom-div-icon',
    iconSize: [24, 24],
    iconAnchor: [12, 12]
  });
  
  return L.marker([node.latitude, node.longitude], { icon });
}

// Icon mapping
const nodeIcons = {
  'bbs_server': '📡',      // Antenna/Tower
  'weather': '🌡️',         // Thermometer
  'mobile': '🚗',          // Car
  'base_station': '🏠',    // House
  'solar': '☀️',           // Sun
  'tdeck': '⌨️',           // Keyboard
  'techo': '📱',           // Phone
  'other': '📡'            // Generic radio
};
```

#### 3.3 Enhanced Tooltips
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

#### 3.4 Updated Legend
**Dynamic legend based on view mode**:

```javascript
function updateLegend(viewMode) {
  const legendDiv = document.querySelector('.legend');
  
  if (viewMode === 'original') {
    // Show role-based colors
    legendDiv.innerHTML = Object.entries(roleColors)
      .map(([role, color]) => `<div><span style="background:${color}"></span>${role}</div>`)
      .join('');
  } else {
    // Show category-based icons
    legendDiv.innerHTML = Object.entries(categoryDisplayNames)
      .map(([category, name]) => `<div><span class="legend-icon">${nodeIcons[category]}</span>${name}</div>`)
      .join('');
  }
}
```

### Phase 4: Styling and UX

#### 4.1 CSS Updates
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

#### 4.2 Responsive Design
- Ensure icons are visible on mobile devices
- Maintain touch-friendly marker sizes
- Preserve existing mobile layout

### Phase 5: Testing and Validation

#### 5.1 Test Cases
- [ ] Toggle between original and classified views
- [ ] Icon rendering on different screen sizes
- [ ] Tooltip accuracy for adjective categories
- [ ] Legend updates correctly
- [ ] Dark mode compatibility
- [ ] Performance with large node counts

#### 5.2 Data Validation
- [ ] Classification accuracy against known nodes
- [ ] Edge cases (nodes with multiple characteristics)
- [ ] Fallback behavior for unclassified nodes

## Node Categories and Icons

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

## Adjective Categories (Tooltip Features)

These are additional characteristics that can be combined with primary categories:

- **☀️ Solar-powered**: Nodes with "solar", "sun", or "☀" in name
- **🚗 Mobile**: Nodes with "mobile", "car", "vehicle" in name  
- **🌡️ Weather**: Nodes with "weather", "temp", "sensor" in name
- **🔋 Battery**: Nodes with low battery status
- **📶 Strong Signal**: Nodes with high signal strength

## Implementation Timeline

### Week 1: Backend Foundation
- [ ] Implement classification algorithm in Ruby
- [ ] Add `/nodes/classified` API endpoint
- [ ] Test classification accuracy

### Week 2: Frontend Core
- [ ] Add map view toggle UI
- [ ] Implement icon-based markers
- [ ] Update legend system

### Week 3: Polish and Testing
- [ ] Enhanced tooltips with adjectives
- [ ] CSS styling and dark mode
- [ ] Mobile responsiveness testing
- [ ] Performance optimization

### Week 4: Validation and Documentation
- [ ] User testing and feedback
- [ ] Documentation updates
- [ ] Final bug fixes

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
