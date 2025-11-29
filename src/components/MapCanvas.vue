<template>
    <div class="relative w-full h-full group">
        <!-- Info Text -->
        <div class="absolute bottom-4 left-1/2 -translate-x-1/2 z-[1000] p-2 bg-white/60 backdrop-blur-md rounded-lg shadow-md text-sm text-gray-700 font-semibold pointer-events-none select-none border border-white/40">
            {{ infoText }}
        </div>

        <!-- Map Container -->
        <div id="map" class="w-full h-full outline-none"></div>

        <!-- Marker Text Input Modal -->
        <div v-if="showMarkerInput" class="absolute inset-0 z-[2000] flex items-center justify-center bg-black/40 backdrop-blur-sm" @click.stop>
            <div class="bg-white p-6 rounded-xl shadow-2xl w-96 animate-fade-in-up">
                <h3 class="text-lg font-bold mb-4 text-gray-800">Add Note to Marker</h3>
                <textarea 
                    v-model="markerText" 
                    maxlength="150"
                    rows="4"
                    class="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none resize-none mb-2"
                    placeholder="Enter description (max 150 chars)..."
                    ref="markerInputRef"
                ></textarea>
                <div class="flex justify-between items-center text-xs text-gray-500 mb-4">
                    <span>{{ markerText.length }}/150</span>
                </div>
                <div class="flex gap-3 justify-end">
                    <button 
                        @click="cancelMarker"
                        class="px-4 py-2 rounded-lg text-gray-600 hover:bg-gray-100 font-medium transition-colors"
                    >
                        Cancel
                    </button>
                    <button 
                        @click="confirmMarker"
                        class="px-4 py-2 rounded-lg bg-blue-600 text-white hover:bg-blue-700 font-medium transition-colors shadow-lg shadow-blue-500/30"
                    >
                        Add Marker
                    </button>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, shallowRef, onMounted, onUnmounted, watch, computed, nextTick, markRaw } from 'vue';
import L from 'leaflet';
import { INITIAL_CENTER, INITIAL_ZOOM, MAX_ZOOM, DrawingTool, TOOL_ICONS } from '../constants.js';

// The map allows zooming beyond the base MAX_ZOOM (18) up to 20.
// We only show popups at the absolute maximum zoom level.
const MAP_MAX_ZOOM = MAX_ZOOM + 2;

// Base size for markers at MAX_ZOOM (Level 18) in pixels
const MARKER_BASE_SIZE_MAX_ZOOM = 5;

// Minimum pixel size for elements to be visible.
// Elements smaller than this will be hidden (opacity 0) to avoid "dust" artifacts at low zoom.
const VISIBILITY_THRESHOLD_PIXELS = 0.1;

const props = defineProps({
  drawings: Array,
  tool: String,
  color: String,
  brushSize: Number,
  user: String,
  userEmail: String,
  isTransparent: Boolean
});

const emit = defineEmits(['add-drawing', 'remove-drawing', 'update-drawing', 'viewport-change']);

const mapInstance = shallowRef(null);
const drawingLayerGroup = shallowRef(null);
const tempLayer = shallowRef(null);

const currentPoints = ref([]);

const isDrawingActive = ref(false);
const isPanning = ref(false);
const lastPanPos = ref(null);
const isErasing = ref(false);
const lastTouchLatLng = ref(null); // Add this for touch handling
const touchStartTimer = ref(null); // Timer for touch delay

const showMarkerInput = ref(false);
const markerText = ref('');
const pendingMarkerPos = ref(null);
const markerInputRef = ref(null);

const infoText = computed(() => {
    if (props.tool === DrawingTool.HAND) return 'Drag to move map';
    if (props.tool === DrawingTool.BRUSH) return 'Hold LEFT button to draw. Hold MIDDLE button to pan.';
    if (props.tool === DrawingTool.MARKER) return 'Click to place marker. Hold MIDDLE button to pan.';
    if (props.tool === DrawingTool.ERASER) return 'Hold LEFT button and drag over items to erase. Hold MIDDLE button to pan.';
    return '';
});

const truncateText = (text, limit) => {
    if (!text) return '';
    return text.length > limit ? text.substring(0, limit) + '...' : text;
};

const createColoredMarkerIcon = (color, size) => {
    const safeSize = Math.max(0.1, size);
    const strokeColor = color.toLowerCase() === '#ffffff' ? '#999999' : 'white';

    const svgHtml = `
        <svg viewBox="0 0 24 24" fill="${color}" stroke="${strokeColor}" stroke-width="1" xmlns="http://www.w3.org/2000/svg" style="filter: drop-shadow(0 1px 1px rgba(0,0,0,0.2)); width: 100%; height: 100%;">
            <path d="${TOOL_ICONS.MARKER.path}" />
        </svg>
    `;
    
    return L.divIcon({
        className: 'custom-colored-marker', 
        html: svgHtml,
        iconSize: [safeSize, safeSize],
        iconAnchor: [safeSize/2, safeSize], 
        popupAnchor: [0, -safeSize],
        tooltipAnchor: [0, -safeSize]
    });
};

// Core logic: Input size is pixel width at reference zoom (default MAX_ZOOM).
// Scale formula: base * 2^(currentZoom - referenceZoom)
const getPhysicalDimension = (baseSize, currentZoom, referenceZoom = MAX_ZOOM) => {
    if (!baseSize) return 0;
    const scale = Math.pow(2, currentZoom - referenceZoom);
    return baseSize * scale;
};

const updateLayerStyles = () => {
    if (!mapInstance.value || !drawingLayerGroup.value) return;
    const currentZoom = mapInstance.value.getZoom();
    const opacity = props.isTransparent ? 0.5 : 1.0;
    const isMaxZoom = currentZoom >= MAP_MAX_ZOOM;
    
    if (!isMaxZoom) {
        mapInstance.value.closePopup();
    }

    drawingLayerGroup.value.eachLayer((layer) => {
        const d = layer.options.drawingData;
        if (!d) return;

        // Case 1: Icon Markers
        if (layer instanceof L.Marker) {
            const currentSize = getPhysicalDimension(MARKER_BASE_SIZE_MAX_ZOOM, currentZoom);
            const isVisible = currentSize >= VISIBILITY_THRESHOLD_PIXELS; // Threshold for visibility

            layer.setIcon(createColoredMarkerIcon(d.color, currentSize));
            const verticalOffset = -currentSize / 5;

            if (layer.getTooltip()) {
                layer.getTooltip().options.offset = [0, verticalOffset];
                if (isMaxZoom) {
                    if (!layer.isTooltipOpen()) layer.openTooltip();
                    const tooltipEl = layer.getTooltip().getElement();
                    if (tooltipEl) tooltipEl.classList.add('zoom-visible');
                } else {
                    layer.closeTooltip();
                }
            }
            
            // Popup Management: Only bind/show at MAX_ZOOM
            if (isMaxZoom) {
                if (!layer.getPopup() && layer.options.customPopupContent) {
                    layer.bindPopup(layer.options.customPopupContent, {
                        ...layer.options.customPopupOptions,
                        offset: [0, verticalOffset]
                    });
                } else if (layer.getPopup()) {
                    layer.getPopup().options.offset = [0, verticalOffset];
                }
            } else {
                if (layer.getPopup()) {
                    layer.unbindPopup();
                }
            }
            
            const iconEl = layer.getElement();
            if (iconEl) iconEl.style.opacity = isVisible ? opacity : 0;
        } 
        // Case 2: Dots (L.Circle - Geographic)
        else if (layer instanceof L.Circle) {
            const baseWeight = d.weight || 1.0;
            const currentPixelSize = getPhysicalDimension(baseWeight, currentZoom, MAX_ZOOM);
            const isVisible = currentPixelSize >= VISIBILITY_THRESHOLD_PIXELS;

             layer.setStyle({ 
                opacity: isVisible ? opacity : 0, 
                fillOpacity: isVisible ? opacity : 0 
            });
        }
        // Case 3: Dots (L.CircleMarker - Pixel based)
        else if (layer instanceof L.CircleMarker) {
            const baseWeight = d.weight || 1.0;
            const currentWeight = getPhysicalDimension(baseWeight, currentZoom, MAX_ZOOM);
            const isVisible = currentWeight >= VISIBILITY_THRESHOLD_PIXELS;

            layer.setRadius(Math.max(0.01, currentWeight / 2));
            layer.setStyle({ 
                opacity: isVisible ? opacity : 0, 
                fillOpacity: isVisible ? opacity : 0 
            });
        }
        // Case 4: Lines (Polyline)
        else if (layer instanceof L.Polyline) {
            const baseWeight = d.weight || 1.0;
            const currentWeight = getPhysicalDimension(baseWeight, currentZoom, MAX_ZOOM);
            const isVisible = currentWeight >= VISIBILITY_THRESHOLD_PIXELS;

            layer.setStyle({ 
                weight: Math.max(0.1, currentWeight), 
                opacity: isVisible ? opacity : 0,
                fillOpacity: isVisible ? opacity : 0
            });
        }
    });

    // Temp Layer Update
    if (tempLayer.value) {
            const currentWeight = getPhysicalDimension(props.brushSize, currentZoom, MAX_ZOOM);
            const isVisible = currentWeight >= VISIBILITY_THRESHOLD_PIXELS;
            const style = { 
                opacity: isVisible ? opacity : 0, 
                fillOpacity: isVisible ? opacity : 0 
            };

            if (tempLayer.value instanceof L.Circle) {
                 tempLayer.value.setStyle(style);
            } else if (tempLayer.value instanceof L.CircleMarker) {
                tempLayer.value.setRadius(Math.max(0.01, currentWeight / 2));
                tempLayer.value.setStyle(style);
            } else if (tempLayer.value instanceof L.Polyline) {
                tempLayer.value.setStyle({ 
                ...style,
                weight: Math.max(0.1, currentWeight)
            });
            }
    }
};

// Expose method to center map
// ...existing code...

// Expose method to center map
const setView = (center, zoom) => {
    if (mapInstance.value) {
        mapInstance.value.setView(center, zoom);
    }
};

defineExpose({ setView });

// ...existing code...
const renderedDrawingIds = new Set();

const renderDrawings = () => {
    if (!drawingLayerGroup.value || !mapInstance.value) return;
    
    // OPTIMIZATION: Incremental Update instead of clearLayers()
    // 1. Remove layers that are no longer in props.drawings
    const currentDataIds = new Set(props.drawings.map(d => d.id));
    const layersToRemove = [];
    
    drawingLayerGroup.value.eachLayer((layer) => {
        const d = layer.options.drawingData;
        if (d && !currentDataIds.has(d.id)) {
            layersToRemove.push(layer);
        }
    });
    
    layersToRemove.forEach(layer => {
        drawingLayerGroup.value.removeLayer(layer);
        if (layer.options.drawingData) {
            renderedDrawingIds.delete(layer.options.drawingData.id);
        }
    });

    const currentZoom = mapInstance.value.getZoom();
    const opacity = props.isTransparent ? 0.5 : 1.0;

    // 2. Add new layers
    props.drawings.forEach(drawing => {
        // Skip if already rendered
        if (renderedDrawingIds.has(drawing.id)) {
            return; 
        }

        try {
            const layer = createLayerForDrawing(drawing, currentZoom, opacity);

            if (layer) {
                const checkErase = (e) => {
                    if (props.tool === DrawingTool.ERASER) {
                        L.DomEvent.stopPropagation(e);
                        const isLeftClickHeld = (e.originalEvent.buttons & 1) === 1;
                        if (isLeftClickHeld || e.type === 'mousedown') {
                                // Check ownership
                                if (drawing.user_id === props.user || drawing.author === props.user || (props.userEmail && drawing.author === props.userEmail)) { // Support both new and old format
                                emit('remove-drawing', drawing.id);
                            }
                        }
                    }
                };
                layer.on('mousedown', checkErase);
                layer.on('mouseover', checkErase);
                
                layer.addTo(drawingLayerGroup.value);
                renderedDrawingIds.add(drawing.id);
                
                // Fix Z-Index: Bring new layers to front
                if (layer.bringToFront) {
                    layer.bringToFront();
                }
            }
        } catch (err) {
            console.error('Error rendering drawing:', drawing.id, err);
        }
    });
    
    nextTick(() => {
        updateLayerStyles();
    });
};

// Helper function to create layer (refactored from renderDrawings)
const createLayerForDrawing = (drawing, currentZoom, opacity) => {
    const currentMarkerSize = getPhysicalDimension(MARKER_BASE_SIZE_MAX_ZOOM, currentZoom);
    const verticalOffset = -currentMarkerSize / 5;
    const isSyncing = drawing.isSyncing;

    let layer;

    if (drawing.tool === DrawingTool.MARKER) {
            const icon = createColoredMarkerIcon(drawing.color, currentMarkerSize);
            // Add glow class to icon if syncing
            if (isSyncing) {
                icon.options.className += ' syncing-glow-marker';
            }

            layer = L.marker(drawing.points[0], {
                icon: icon,
                drawingData: drawing
            });
            
            if (drawing.text) {
            const displayText = truncateText(drawing.text, 7);
            const tooltipContent = `<div class="tooltip-inner-content">${displayText}</div>`;
            layer.bindTooltip(tooltipContent, { 
                permanent: true, 
                direction: 'top', 
                offset: [0, verticalOffset],
                className: 'marker-tooltip-wrapper'
            });
            }

            let dateObj = new Date(drawing.created_at || drawing.id); // Use created_at if available
            if (isNaN(dateObj.getTime())) dateObj = new Date();
            const timeStr = `${dateObj.getFullYear()}-${String(dateObj.getMonth() + 1).padStart(2, '0')}-${String(dateObj.getDate()).padStart(2, '0')} ${String(dateObj.getHours()).padStart(2, '0')}:${String(dateObj.getMinutes()).padStart(2, '0')}`;
            
            const popupContent = `
            <div class="px-2 py-1 min-w-[80px] max-w-[180px]">
                <div class="font-bold text-[10px] text-gray-900 mb-0.5">${drawing.user_name || drawing.author || 'Unknown'}</div>
                <div class="text-[10px] font-medium text-gray-800 mb-0.5 whitespace-normal break-words leading-tight">${drawing.text || ''}</div>
                <div class="text-[9px] text-gray-400">${timeStr}</div>
                ${isSyncing ? '<div class="text-[9px] text-amber-500 font-bold mt-1">Saving...</div>' : ''}
            </div>
            `;

            const popupOptions = {
                closeButton: false,
                offset: [0, verticalOffset],
                className: 'custom-popup marker-popup-custom'
            };

            layer.options.customPopupContent = popupContent;
            layer.options.customPopupOptions = popupOptions;

            layer.bindPopup(popupContent, popupOptions);

            let closeTimer = null;
            
            // Click handler for mobile/tablet support and explicit interaction
            layer.on('click', function(e) {
                L.DomEvent.stopPropagation(e);
                if (closeTimer) {
                    clearTimeout(closeTimer);
                    closeTimer = null;
                }
                this.openPopup();
                
                // Add visual feedback
                requestAnimationFrame(() => {
                    const popup = this.getPopup();
                    if (popup) {
                        const el = popup.getElement();
                        if (el) el.classList.add('popup-visible');
                    }
                });
            });

            layer.on('mouseover', function() {
            if (mapInstance.value.getZoom() < MAP_MAX_ZOOM) return;
            if (closeTimer) { clearTimeout(closeTimer); closeTimer = null; }
            const tooltip = this.getTooltip();
            if (tooltip) {
                const el = tooltip.getElement();
                if (el) el.classList.add('hover-hidden');
            }
            this.openPopup();
            requestAnimationFrame(() => {
                const popup = this.getPopup();
                if (popup) {
                    const el = popup.getElement();
                    if (el) el.classList.add('popup-visible');
                }
            });
            });

            layer.on('mouseout', function() {
                const popup = this.getPopup();
                const tooltip = this.getTooltip();
                if (popup) {
                    const el = popup.getElement();
                    if (el) el.classList.remove('popup-visible');
                }
                closeTimer = setTimeout(() => {
                    this.closePopup();
                    if (tooltip) {
                        const el = tooltip.getElement();
                        if (el && mapInstance.value.getZoom() >= MAP_MAX_ZOOM) {
                            el.classList.remove('hover-hidden');
                        }
                    }
                }, 300); 
            });
    } 
    else if (drawing.tool === DrawingTool.BRUSH) {
        // Calculate width at current zoom - Use MAX_ZOOM as unified reference
        const currentWeight = getPhysicalDimension(drawing.weight || 1.0, currentZoom, MAX_ZOOM);
        const pathOptions = { 
            color: drawing.color, 
            // Ensure consistent visual properties
            weight: Math.max(0.1, currentWeight),
            lineCap: 'round', 
            lineJoin: 'round',
            opacity: opacity, 
            fillOpacity: opacity,
            drawingData: drawing,
            className: isSyncing ? 'syncing-glow-path' : '' // Add glow class for paths
        };

        if (drawing.points.length === 1) {
            // Single point: Use L.Circle for smooth geographic scaling
            
            // FIX: Calculate meters directly at MAX_ZOOM to avoid low-zoom pixel clamping issues.
            // This ensures the physical size (meters) remains constant regardless of the zoom level at render time.
            const baseWeight = drawing.weight || 1.0;
            const lat = drawing.points[0].lat || drawing.points[0][0]; // Handle object or array format
            
            // Web Mercator resolution formula: meters/pixel = (EarthCircumference * cos(lat)) / (256 * 2^zoom)
            const earthCircumference = 40075016.686;
            const latRad = lat * (Math.PI / 180);
            const resolutionAtMaxZoom = (earthCircumference * Math.cos(latRad)) / (256 * Math.pow(2, MAX_ZOOM));
            
            // Radius (meters) = (Base Pixel Radius) * Resolution
            let radiusMeters = (baseWeight / 2) * resolutionAtMaxZoom;
            
            // Fallback purely for safety
            if (!radiusMeters || radiusMeters <= 0) {
                radiusMeters = 0.1; 
            }

            layer = L.circle(drawing.points[0], {
                ...pathOptions,
                stroke: false,
                radius: radiusMeters, 
            });
        } else {
            // Multiple points: Polyline
            layer = L.polyline(drawing.points, pathOptions);
        }
    }
    return layer;
};

const renderTempDrawing = () => {
    if (!mapInstance.value) return;

    if (tempLayer.value) {
        tempLayer.value.remove();
        tempLayer.value = null;
    }
    if (currentPoints.value.length === 0) return;

    const currentZoom = mapInstance.value.getZoom();
    // Consistent Width Calculation - Use MAX_ZOOM as reference
    const currentWeight = getPhysicalDimension(props.brushSize, currentZoom, MAX_ZOOM);

    const pathOptions = { 
        color: props.color, 
        weight: Math.max(0.1, currentWeight),
        lineCap: 'round',
        lineJoin: 'round',
        opacity: props.isTransparent ? 0.5 : 1,
        fillOpacity: props.isTransparent ? 0.5 : 1,
    };
    
    if (props.tool === DrawingTool.BRUSH) {
        if (currentPoints.value.length === 1) {
            // Dot: Use L.Circle
            // FIX: Calculate meters directly at MAX_ZOOM to avoid low-zoom pixel clamping issues.
            const lat = currentPoints.value[0].lat || currentPoints.value[0][0];
            const earthCircumference = 40075016.686;
            const latRad = lat * (Math.PI / 180);
            const resolutionAtMaxZoom = (earthCircumference * Math.cos(latRad)) / (256 * Math.pow(2, MAX_ZOOM));
            
            let radiusMeters = (props.brushSize / 2) * resolutionAtMaxZoom;
            
            if (!radiusMeters || radiusMeters <= 0) radiusMeters = 0.1;

            tempLayer.value = L.circle(currentPoints.value[0], {
                ...pathOptions,
                stroke: false,
                radius: radiusMeters
            }).addTo(mapInstance.value);
        } else {
            // Line: Weight = Width
            tempLayer.value = L.polyline(currentPoints.value, pathOptions).addTo(mapInstance.value);
        }
    }
    
    // Fix Z-Index: Ensure temp layer is always on top
    if (tempLayer.value && tempLayer.value.bringToFront) {
        tempLayer.value.bringToFront();
    }
};


const finishDrawing = () => {
    if (!mapInstance.value) return;
    if (currentPoints.value.length === 0) return;
    
    emit('add-drawing', {
        tool: props.tool,
        color: props.color,
        weight: props.brushSize, 
        points: [...currentPoints.value],
        createdZoom: Math.round(mapInstance.value.getZoom()) 
    });
    
    currentPoints.value = [];
    renderTempDrawing();
};

const cancelMarker = () => {
    showMarkerInput.value = false;
    markerText.value = '';
    pendingMarkerPos.value = null;
};

const confirmMarker = () => {
    if (!pendingMarkerPos.value || !mapInstance.value) return;
    emit('add-drawing', {
        tool: DrawingTool.MARKER,
        color: props.color,
        weight: 0, 
        points: [pendingMarkerPos.value],
        text: markerText.value,
        createdZoom: Math.round(mapInstance.value.getZoom())
    });
    cancelMarker();
};

// Pointer Event Handlers (Unified for Mouse/Touch/Pen)
const handlePointerDown = (e) => {
    if (props.tool === DrawingTool.HAND) return; // Let Leaflet handle panning
    
    // Allow multi-touch gestures to pass through if not drawing
    if (!e.isPrimary) return;

    // For mouse, only allow left click
    if (e.pointerType === 'mouse' && e.button !== 0) return;

    // Prevent default to stop scrolling/zooming while drawing
    // Note: touch-action: none in CSS handles most of this, but preventDefault is safe
    // e.preventDefault(); 
    // WARNING: preventDefault() on pointerdown might block focus or other behaviors. 
    // Rely on touch-action: none for scrolling.
    
    e.stopPropagation(); // Stop Leaflet from panning

    const container = mapInstance.value.getContainer();
    container.setPointerCapture(e.pointerId);

    const rect = container.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    const latlng = mapInstance.value.containerPointToLatLng([x, y]);

    if (props.tool === DrawingTool.BRUSH) {
        isDrawingActive.value = true;
        currentPoints.value = [[latlng.lat, latlng.lng]];
        renderTempDrawing();
    } else if (props.tool === DrawingTool.ERASER) {
        isErasing.value = true;
        checkEraserHit(latlng);
    } else if (props.tool === DrawingTool.MARKER) {
        // Marker logic is usually on 'click' (up), but we can track down position
        pendingMarkerPos.value = [latlng.lat, latlng.lng];
    }
};

const handlePointerMove = (e) => {
    if (props.tool === DrawingTool.HAND) return;
    if (!isDrawingActive.value && !isErasing.value) return;

    e.preventDefault();
    e.stopPropagation();

    // Use getCoalescedEvents to capture high-frequency input (fast movement)
    const events = e.getCoalescedEvents ? e.getCoalescedEvents() : [e];
    const container = mapInstance.value.getContainer();
    const rect = container.getBoundingClientRect();

    if (isDrawingActive.value && props.tool === DrawingTool.BRUSH) {
        events.forEach(evt => {
            const x = evt.clientX - rect.left;
            const y = evt.clientY - rect.top;
            const latlng = mapInstance.value.containerPointToLatLng([x, y]);
            currentPoints.value.push([latlng.lat, latlng.lng]);
        });

        if (tempLayer.value && mapInstance.value) {
            if (tempLayer.value instanceof L.Polyline && !(tempLayer.value instanceof L.CircleMarker) && !(tempLayer.value instanceof L.Circle)) {
                try {
                    tempLayer.value.setLatLngs(currentPoints.value);
                } catch (err) {
                    renderTempDrawing();
                }
            } else {
                renderTempDrawing();
            }
        }
    } else if (isErasing.value && props.tool === DrawingTool.ERASER) {
        // Eraser doesn't need coalesced events as much, but good to have
        events.forEach(evt => {
            const x = evt.clientX - rect.left;
            const y = evt.clientY - rect.top;
            const latlng = mapInstance.value.containerPointToLatLng([x, y]);
            checkEraserHit(latlng);
        });
    }
};

const handlePointerUp = (e) => {
    if (props.tool === DrawingTool.HAND) return;

    const container = mapInstance.value.getContainer();
    if (container.hasPointerCapture(e.pointerId)) {
        container.releasePointerCapture(e.pointerId);
    }

    if (isDrawingActive.value) {
        isDrawingActive.value = false;
        finishDrawing();
    }
    if (isErasing.value) {
        isErasing.value = false;
    }
    
    if (props.tool === DrawingTool.MARKER && pendingMarkerPos.value) {
        // Confirm marker on up
        markerText.value = ''; 
        showMarkerInput.value = true;
        nextTick(() => {
            if (markerInputRef.value) markerInputRef.value.focus();
        });
    }
};

const updateMapInteraction = () => {
    if (!mapInstance.value) return;
    const container = mapInstance.value.getContainer();

    if (props.tool === DrawingTool.HAND) {
        mapInstance.value.dragging.enable();
        container.style.cursor = 'grab';
        container.style.touchAction = 'auto';
    } else {
        mapInstance.value.dragging.disable(); 
        if (props.tool === DrawingTool.ERASER) {
            container.style.cursor = 'not-allowed'; 
        } else {
            container.style.cursor = 'crosshair';
        }
        container.style.touchAction = 'none';
    }
};

// ...existing code...
// Helper: Solve quadratic equation to find intersection of line segment and circle
// Returns array of t values [0..1] where intersection occurs
const getSegmentCircleIntersections = (p1, p2, center, radius) => {
    const dx = p2.x - p1.x;
    const dy = p2.y - p1.y;
    const A = dx * dx + dy * dy;
    const B = 2 * (dx * (p1.x - center.x) + dy * (p1.y - center.y));
    const C = (p1.x - center.x) ** 2 + (p1.y - center.y) ** 2 - radius * radius;

    const det = B * B - 4 * A * C;
    if (det < 0) return []; // No intersection

    const t1 = (-B - Math.sqrt(det)) / (2 * A);
    const t2 = (-B + Math.sqrt(det)) / (2 * A);

    const intersections = [];
    if (t1 >= 0 && t1 <= 1) intersections.push(t1);
    if (t2 >= 0 && t2 <= 1) intersections.push(t2);

    return intersections.sort((a, b) => a - b);
};

// Touch Event Handlers
const checkEraserHit = (latlng) => {
    if (!mapInstance.value || !drawingLayerGroup.value) return;
    
    const eraserPoint = mapInstance.value.latLngToContainerPoint(latlng);
    const currentZoom = mapInstance.value.getZoom();
    
    // Eraser radius in pixels (Fixed size, approx equivalent to brush size 1.5)
    const eraserRadius = getPhysicalDimension(30, currentZoom, MAX_ZOOM); 

    drawingLayerGroup.value.eachLayer((layer) => {
        const d = layer.options.drawingData;
        if (!d) return;
        
        // Check ownership
        const isOwner = d.user_id === props.user || d.author === props.user || (props.userEmail && d.author === props.userEmail);
        if (!isOwner) return;

        // 1. Handle Markers/Dots (Simple Distance Check)
        if (layer instanceof L.Marker || layer instanceof L.CircleMarker || layer instanceof L.Circle) {
            const layerPoint = mapInstance.value.latLngToContainerPoint(layer.getLatLng());
            if (layerPoint.distanceTo(eraserPoint) < eraserRadius) { 
                emit('remove-drawing', d.id);
            }
        } 
        // 2. Handle Polylines (Stroke-based Eraser)
        else if (layer instanceof L.Polyline) {
            const latlngs = layer.getLatLngs();
            // Handle nested arrays (Leaflet can return [[lat,lng],...] for simple polylines)
            const points = (latlngs.length > 0 && Array.isArray(latlngs[0])) ? latlngs.flat() : latlngs;
            
            if (points.length < 2) return;

            // Convert all points to pixels for calculation
            const pixelPoints = points.map(p => mapInstance.value.latLngToContainerPoint(p));
            
            let hit = false;

            // 1. Check if any point is inside eraser
            if (pixelPoints.some(p => p.distanceTo(eraserPoint) < eraserRadius)) {
                hit = true;
            } 
            // 2. Check if any segment intersects eraser
            else {
                for (let i = 0; i < pixelPoints.length - 1; i++) {
                    const p1 = pixelPoints[i];
                    const p2 = pixelPoints[i+1];
                    const ts = getSegmentCircleIntersections(p1, p2, eraserPoint, eraserRadius);
                    if (ts.length > 0) {
                        hit = true;
                        break;
                    }
                }
            }

            if (hit) {
                emit('remove-drawing', d.id);
            }
        }
    });
};

// Removed old Touch Handlers (handleTouchStart, handleTouchMove, handleTouchEnd) as they are replaced by Pointer Events


onMounted(() => {
    // ...existing code...
    // Try to load saved view state
    let initialCenter = INITIAL_CENTER;
    let initialZoom = INITIAL_ZOOM;
    
    try {
        const savedView = localStorage.getItem('geoCanvasViewState');
        if (savedView) {
            const parsed = JSON.parse(savedView);
            if (parsed.center && parsed.zoom) {
                initialCenter = parsed.center;
                initialZoom = parsed.zoom;
            }
        }
    } catch (e) {
        console.warn('Failed to load saved view state', e);
    }

    const map = markRaw(L.map('map', {
        center: initialCenter,
        zoom: initialZoom,
        zoomControl: false,
        attributionControl: false,
        renderer: L.canvas({ padding: 0.5 }),
        maxZoom: MAP_MAX_ZOOM
    }));
    
    mapInstance.value = map;
    
    L.control.zoom({ position: 'bottomright' }).addTo(map);
    L.control.attribution({ position: 'bottomright' }).addTo(map);

    L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
        attribution: '&copy; OpenStreetMap, &copy; CARTO',
        subdomains: 'abcd',
        maxNativeZoom: 19, // Optimization: CartoDB supports up to level 19
        maxZoom: MAP_MAX_ZOOM 
    }).addTo(map);

    drawingLayerGroup.value = markRaw(L.layerGroup().addTo(map));
    
    // Optimization: Update styles only on zoomend to allow "auto-stretch" behavior during animation
    // and prevent heavy re-calculations on every frame.
    map.on('zoomend', updateLayerStyles);
    
    // Save view state on moveend
    map.on('moveend', () => {
        const center = map.getCenter();
        const zoom = map.getZoom();
        localStorage.setItem('geoCanvasViewState', JSON.stringify({
            center: [center.lat, center.lng],
            zoom: zoom
        }));

        // Emit viewport change
        const bounds = map.getBounds();
        emit('viewport-change', {
            minLat: bounds.getSouth(),
            maxLat: bounds.getNorth(),
            minLng: bounds.getWest(),
            maxLng: bounds.getEast(),
            zoom: zoom
        });
    });
    
    // Use Pointer Events for unified and high-frequency input handling (Fixes fast movement issue)
    const container = map.getContainer();
    container.addEventListener('pointerdown', handlePointerDown);
    container.addEventListener('pointermove', handlePointerMove);
    container.addEventListener('pointerup', handlePointerUp);
    container.addEventListener('pointercancel', handlePointerUp);
    container.addEventListener('pointerleave', handlePointerUp);

    renderDrawings();
    updateMapInteraction();

    // Fix for map not displaying fully on initial load
    setTimeout(() => {
        if (mapInstance.value) {
            mapInstance.value.invalidateSize();
            
            // Initial viewport emit
            const bounds = mapInstance.value.getBounds();
            emit('viewport-change', {
                minLat: bounds.getSouth(),
                maxLat: bounds.getNorth(),
                minLng: bounds.getWest(),
                maxLng: bounds.getEast(),
                zoom: mapInstance.value.getZoom()
            });
        }
    }, 100);
});

onUnmounted(() => {
    if (mapInstance.value) {
        // Remove Pointer Listeners
        const container = mapInstance.value.getContainer();
        container.removeEventListener('pointerdown', handlePointerDown);
        container.removeEventListener('pointermove', handlePointerMove);
        container.removeEventListener('pointerup', handlePointerUp);
        container.removeEventListener('pointercancel', handlePointerUp);
        container.removeEventListener('pointerleave', handlePointerUp);

        mapInstance.value.remove();
        mapInstance.value = null;
    }
});


watch(() => props.drawings, renderDrawings, { deep: true });

watch(() => props.tool, () => {
    currentPoints.value = [];
    renderTempDrawing();
    updateMapInteraction();
});

watch(() => props.brushSize, () => {
    renderTempDrawing();
});

watch(() => props.isTransparent, () => {
    renderTempDrawing();
    updateLayerStyles();
});
</script>

<style>
/* 
    CUSTOM TOOLTIP STYLING 
*/
.marker-tooltip-wrapper {
    background: transparent !important;
    border: none !important;
    box-shadow: none !important;
    padding: 0 !important;
}
.marker-tooltip-wrapper::before { display: none; }

.marker-tooltip-wrapper .tooltip-inner-content {
    background-color: rgba(255, 255, 255, 0.9);
    backdrop-filter: blur(4px);
    border: 1px solid #d1d5db;
    border-radius: 0.75rem;
    box-shadow: 0 1px 2px rgba(0,0,0,0.1);
    padding: 2px 8px;
    font-size: 10px;
    font-weight: 500;
    color: #1f2937;
    white-space: nowrap;
    transform-origin: bottom center;
    transition: transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1), opacity 0.3s ease;
    transform: scale(0);
    opacity: 0;
}

.marker-tooltip-wrapper.zoom-visible .tooltip-inner-content {
    transform: scale(1);
    opacity: 1;
}

.marker-tooltip-wrapper.hover-hidden .tooltip-inner-content {
    transform: scale(0);
    opacity: 0;
}

/* 
    CUSTOM POPUP STYLING 
*/
.marker-popup-custom .leaflet-popup-content-wrapper {
    background-color: rgba(255, 255, 255, 0.9) !important;
    backdrop-filter: blur(4px);
    border: 1px solid #d1d5db !important;
    border-radius: 0.75rem !important;
    box-shadow: 0 1px 2px rgba(0,0,0,0.1) !important;
    padding: 0 !important; 
    transition: transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1), opacity 0.3s ease;
    transform-origin: bottom center;
    transform: scale(0);
    opacity: 0;
}

.marker-popup-custom .leaflet-popup-content {
    margin: 0 !important;
    width: auto !important;
    line-height: 1.2;
}

.marker-popup-custom .leaflet-popup-tip-container {
    display: none;
}

.marker-popup-custom.popup-visible .leaflet-popup-content-wrapper {
    transform: scale(1);
    opacity: 1;
}

/* Syncing Glow Effects */
@keyframes pulse-gold {
    0% { filter: drop-shadow(0 0 2px gold); }
    50% { filter: drop-shadow(0 0 8px gold); }
    100% { filter: drop-shadow(0 0 2px gold); }
}

.syncing-glow-marker {
    animation: pulse-gold 1.5s infinite;
}

/* For SVG Paths (Polylines) */
.leaflet-overlay-pane path.syncing-glow-path {
    stroke: gold !important; /* Override color temporarily */
    stroke-opacity: 0.8;
    filter: drop-shadow(0 0 4px gold);
    animation: pulse-gold-path 1.5s infinite;
}

@keyframes pulse-gold-path {
    0% { stroke-width: inherit; filter: drop-shadow(0 0 2px gold); }
    50% { stroke-width: inherit; filter: drop-shadow(0 0 6px gold); }
    100% { stroke-width: inherit; filter: drop-shadow(0 0 2px gold); }
}
</style>