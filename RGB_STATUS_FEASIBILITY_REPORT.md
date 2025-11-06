# RGB Status Feature Feasibility Report for Imprint Keyboard

## Executive Summary

This report analyzes the feasibility of implementing an RGB status feature for the Cyboard Imprint keyboard, similar to the MoErgo Glove80's RGB status macro. The feature would use RGB LEDs to display keyboard status information including battery levels, Bluetooth connections, active layers, and lock indicators.

**Key Finding:** The Imprint keyboard **HAS** RGB underglow hardware support and can implement RGB status features, but it requires creating a custom implementation since this feature does not exist in vanilla ZMK.

---

## 1. Current Hardware Configuration

### 1.1 Imprint Keyboard Setup

**Board:** `assimilator-bt` (Bluetooth controller board from Cyboard)
**Shield:** `imprint_left` and `imprint_right` (split keyboard configuration)
**Firmware:** Vanilla ZMK from `zmkfirmware/zmk` (main branch)

### 1.2 RGB Hardware Status

✅ **RGB Underglow: ENABLED**

Based on `/config/imprint.conf`:
```conf
CONFIG_ZMK_RGB_UNDERGLOW=y
CONFIG_WS2812_STRIP=y
```

The keyboard has:
- RGB underglow capability using WS2812 LED strips
- RGB controls already configured in the Magic layer (layer 4)
- Standard RGB commands: toggle, brightness, saturation, hue, speed, effects

**Finding:** The Imprint keyboard HAS the necessary RGB LED hardware to implement status display features.

### 1.3 Unknown Hardware Specifications

⚠️ **Information Not Found:**
- Exact number of RGB LEDs per side
- Physical LED layout/positioning
- LED chain configuration (serial order)

**Impact:** LED mapping will need to be determined empirically or from hardware documentation.

---

## 2. RGB Status Feature Architecture

### 2.1 Glove80 Implementation Overview

The MoErgo Glove80 implements RGB status as a **custom ZMK fork feature**:

**Architecture:**
- Adds `RGB_STATUS` command (value 15) beyond vanilla ZMK's commands (0-14)
- Modifies `dt-bindings/zmk/rgb.h` to define the new command
- Implements case handler in `behavior_rgb_underglow.c`
- Core display logic in `rgb_underglow.c`

**Display Mechanism:**
- Activates for 10 seconds
- Updates every 25ms (~400 animation steps total)
- Smooth fade-in/fade-out animation
- Automatically returns to previous RGB state

### 2.2 Vanilla ZMK RGB Commands

Current vanilla ZMK supports commands 0-14:

| Command | Value | Function |
|---------|-------|----------|
| RGB_TOG_CMD | 0 | Toggle on/off |
| RGB_ON_CMD | 1 | Turn on |
| RGB_OFF_CMD | 2 | Turn off |
| RGB_HUI_CMD | 3 | Hue increase |
| RGB_HUD_CMD | 4 | Hue decrease |
| RGB_SAI_CMD | 5 | Saturation increase |
| RGB_SAD_CMD | 6 | Saturation decrease |
| RGB_BRI_CMD | 7 | Brightness increase |
| RGB_BRD_CMD | 8 | Brightness decrease |
| RGB_SPI_CMD | 9 | Speed increase |
| RGB_SPD_CMD | 10 | Speed decrease |
| RGB_EFF_CMD | 11 | Next effect |
| RGB_EFR_CMD | 12 | Previous effect |
| RGB_EFS_CMD | 13 | Set effect |
| RGB_COLOR_HSB_CMD | 14 | Set HSB color |

**Gap:** Command value 15 (RGB_STATUS) is available for custom implementation.

---

## 3. Available ZMK APIs for Status Information

### 3.1 Battery Status ✅ **AVAILABLE**

**API:** Standard ZMK battery functions

```c
uint8_t zmk_battery_state_of_charge(void);
```

**Capabilities:**
- Returns battery percentage (0-100)
- Works on central side
- Real-time updates available

**Status:** ✅ Fully supported for local battery

### 3.2 Peripheral Battery Status ⚠️ **PARTIALLY AVAILABLE**

**Configuration Required:**
```conf
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y
```

**Challenges:**
- API function not clearly documented in public headers
- May require custom implementation or event subscription
- The Glove80 documentation mentions `zmk_split_central_get_peripheral_battery_level()` but this function was not found in vanilla ZMK

**Status:** ⚠️ Requires investigation; may need custom implementation

### 3.3 Bluetooth Profile Status ✅ **AVAILABLE**

**API:** From `app/include/zmk/ble.h`

```c
// Profile queries
int zmk_ble_active_profile_index(void);
bool zmk_ble_active_profile_is_open(void);
bool zmk_ble_active_profile_is_connected(void);
bool zmk_ble_profile_is_connected(uint8_t index);
bool zmk_ble_profile_is_open(uint8_t index);
```

**Capabilities:**
- Query active profile (0-4)
- Check connection status per profile
- Check if profile is paired/open

**Status:** ✅ Fully supported

### 3.4 USB Connection Status ✅ **AVAILABLE**

**API:** From `app/include/zmk/usb.h`

```c
enum zmk_usb_conn_state zmk_usb_get_conn_state(void);
bool zmk_usb_is_powered(void);
bool zmk_usb_is_hid_ready(void);
```

**States:**
- `ZMK_USB_CONN_NONE` - Disconnected
- `ZMK_USB_CONN_POWERED` - Powered but not connected
- `ZMK_USB_CONN_HID` - Connected and ready

**Status:** ✅ Fully supported

### 3.5 Output Selection (USB vs BLE) ✅ **AVAILABLE**

**API:** From `app/include/zmk/endpoints.h`

```c
struct zmk_endpoint_instance zmk_endpoints_selected(void);
```

**Capabilities:**
- Determine current output (USB or BLE)
- Get endpoint information

**Status:** ✅ Fully supported

### 3.6 HID Indicators (Lock Keys) ✅ **AVAILABLE**

**API:** From `app/include/zmk/hid_indicators.h`

```c
zmk_hid_indicators_t zmk_hid_indicators_get_current_profile(void);
```

**Capabilities:**
- Caps Lock status
- Num Lock status
- Scroll Lock status

**Requirement:** Must have `CONFIG_ZMK_HID_INDICATORS=y`

**Status:** ✅ Fully supported

### 3.7 Layer Status ✅ **AVAILABLE**

**API:** From `app/include/zmk/keymap.h`

```c
bool zmk_keymap_layer_active(zmk_keymap_layer_id_t layer);
zmk_keymap_layer_index_t zmk_keymap_highest_layer_active(void);
zmk_keymap_layers_state_t zmk_keymap_layer_state(void);
```

**Capabilities:**
- Check if specific layer is active
- Get highest active layer
- Get full layer state bitmap

**Status:** ✅ Fully supported

---

## 4. Feasibility Assessment by Feature

### 4.1 Battery Level Display

**Feasibility:** ✅ **HIGH** (95% confident)

**Requirements:**
- Query: `zmk_battery_state_of_charge()`
- Display: Color-coded LEDs based on percentage
  - Green: >40%
  - Yellow: 20-40%
  - Red: ≤20%

**Implementation Notes:**
- Straightforward API access
- Need to map battery percentage to LED positions
- Can spread across multiple LEDs for visual effect

**Recommendation:** Implement as Phase 1 priority feature

---

### 4.2 Split Keyboard Peripheral Battery

**Feasibility:** ⚠️ **MEDIUM** (60% confident)

**Challenges:**
1. API function mentioned in Glove80 docs not found in vanilla ZMK
2. May require:
   - Custom implementation
   - Event subscription mechanism
   - Direct GATT characteristic reading

**Requirements:**
```conf
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y
```

**Potential Solutions:**
- Option A: Find undocumented API in ZMK source
- Option B: Subscribe to battery change events
- Option C: Implement custom peripheral query
- Option D: Skip peripheral battery, show only central

**Recommendation:**
- Implement Option D initially (central battery only)
- Investigate Options A-C for Phase 2 enhancement

---

### 4.3 HID Lock Indicators (Caps/Num/Scroll Lock)

**Feasibility:** ✅ **HIGH** (90% confident)

**Requirements:**
- Configuration: `CONFIG_ZMK_HID_INDICATORS=y`
- Query: `zmk_hid_indicators_get_current_profile()`
- Display: Dedicated LED(s) in red when active

**Implementation Notes:**
- Clean API available
- Typically shown on specific LED positions
- Red color standard for lock indicators

**Recommendation:** Implement as Phase 2 feature

---

### 4.4 Layer Status Display

**Feasibility:** ✅ **HIGH** (95% confident)

**Requirements:**
- Query: `zmk_keymap_layer_active()` or `zmk_keymap_highest_layer_active()`
- Display: Magenta/purple LEDs for active layers

**Implementation Notes:**
- Very straightforward API
- Current keymap has 6 layers (Base through AutoMouse)
- Can show either:
  - All active layers simultaneously
  - Only highest active layer

**Recommendation:** Implement as Phase 1 priority feature

---

### 4.5 Bluetooth Profile Status

**Feasibility:** ✅ **VERY HIGH** (98% confident)

**Requirements:**
- APIs all available in `zmk/ble.h`
- Display scheme (per Glove80):
  - White: Active profile
  - Dull green: Paired and connected (inactive)
  - Red: Paired but disconnected
  - Lilac/purple: Unpaired slot

**Implementation Notes:**
- Excellent API coverage
- Supports up to 5 BLE profiles (ZMK standard)
- Can dedicate 5 LEDs for profile status

**Recommendation:** Implement as Phase 1 priority feature

---

### 4.6 USB Connection Status

**Feasibility:** ✅ **VERY HIGH** (98% confident)

**Requirements:**
- Query: `zmk_usb_get_conn_state()`
- Display: Single LED showing USB state
  - Blue/cyan: Connected (HID ready)
  - Dim blue: Powered only
  - Off: Disconnected

**Implementation Notes:**
- Simple API with clear states
- Can combine with output selection display

**Recommendation:** Implement as Phase 1 priority feature

---

### 4.7 Fade Animation

**Feasibility:** ✅ **HIGH** (85% confident)

**Requirements:**
- Zephyr k_timer API for periodic updates
- RGB blending calculations
- State management for animation

**Implementation Notes:**
- 25ms update interval (Glove80 standard)
- Can extend to 50ms if performance is an issue
- Need to save/restore previous RGB state
- Fade-in and fade-out curves

**Recommendation:** Implement as Phase 3 polish feature

---

## 5. Implementation Approach

### 5.1 Required Code Modifications

**1. Header File: `dt-bindings/zmk/rgb.h`**
```c
// Add after RGB_COLOR_HSB_CMD (14)
#define RGB_STATUS_CMD 15
#define RGB_STATUS RGB_STATUS_CMD
```

**2. Behavior File: `behavior_rgb_underglow.c`**
- Add case handler for `RGB_STATUS_CMD`
- Call status display function

**3. Core Implementation: `rgb_underglow.c`**
- New function: `rgb_underglow_status_display()`
- Status query logic
- LED mapping
- Timer management
- Animation blending

**4. Configuration: Enable required features**
```conf
CONFIG_ZMK_HID_INDICATORS=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y
```

### 5.2 Implementation Strategy

**Phase 1: Foundation & Core Features**
- Add RGB_STATUS command infrastructure
- Implement timer and basic display mechanism
- Add features:
  - ✅ Battery level (central)
  - ✅ Layer status
  - ✅ Bluetooth profile status
  - ✅ USB connection status

**Phase 2: Enhanced Status**
- Add features:
  - ✅ HID lock indicators
  - ⚠️ Peripheral battery (if API found)

**Phase 3: Polish**
- Implement fade-in/fade-out animation
- Optimize LED mapping
- Performance tuning
- Battery usage optimization

**Phase 4: Customization** (Optional)
- Device tree properties for:
  - Display duration
  - Update interval
  - LED position mapping
  - Color customization

### 5.3 Development Environment

**Options:**

**Option A: User Config Repository (Current)**
- ❌ Cannot modify ZMK core files
- ❌ Cannot add new commands to RGB behavior
- **Verdict:** Not suitable for this feature

**Option B: ZMK Module**
- ✅ Can extend behaviors
- ⚠️ Limited - cannot modify core behaviors
- ⚠️ Complex setup
- **Verdict:** Possible but difficult

**Option C: Custom ZMK Fork**
- ✅ Full control over ZMK source
- ✅ Can modify any file
- ✅ Can merge upstream updates
- ❌ Requires maintaining a fork
- **Verdict:** Recommended approach

**Recommendation:** Create a custom ZMK fork for the Imprint keyboard, similar to how MoErgo maintains their Glove80 fork.

---

## 6. LED Mapping Considerations

### 6.1 Unknown LED Configuration

The exact LED layout is not documented. Need to determine:
- Total LED count per side
- Physical LED positions
- Serial chain order

### 6.2 Proposed Mapping Strategy

**Generic allocation for ~10-20 LEDs per side:**

```
Position     | Status Information
-------------|-------------------
LEDs 0-2     | Battery level (central)
LEDs 3-4     | Battery level (peripheral) [if available]
LED 5        | Caps Lock
LED 6        | Num Lock
LED 7        | Scroll Lock
LEDs 8-12    | Bluetooth profiles 0-4
LED 13       | USB connection
LEDs 14-19   | Active layers / reserved
```

**Note:** Actual mapping depends on hardware and will need adjustment.

---

## 7. Estimated Effort

### 7.1 Development Time Estimates

**Phase 1 (Core Features):**
- Setup fork and build environment: 2-4 hours
- Implement RGB_STATUS command: 4-6 hours
- Implement core status queries: 6-8 hours
- Testing and debugging: 4-6 hours
- **Total Phase 1: 16-24 hours**

**Phase 2 (Enhanced Status):**
- HID indicators: 2-3 hours
- Peripheral battery investigation: 4-8 hours
- Testing: 2-4 hours
- **Total Phase 2: 8-15 hours**

**Phase 3 (Polish):**
- Animation system: 4-6 hours
- LED mapping refinement: 2-4 hours
- Optimization: 2-4 hours
- **Total Phase 3: 8-14 hours**

**Total Estimated Effort: 32-53 hours**

### 7.2 Technical Complexity

**Overall Complexity: MODERATE**

**Skills Required:**
- C programming (intermediate)
- ZMK firmware architecture understanding
- Device tree basics
- Zephyr RTOS familiarity (timers, events)
- Git/GitHub workflow

**Learning Curve:**
- Medium if familiar with embedded systems
- Medium-High if new to ZMK/Zephyr

---

## 8. Risks and Mitigation

### 8.1 Technical Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Peripheral battery API unavailable | Medium | Implement central-only initially |
| LED count/mapping unknown | Low | Empirical testing, flexible mapping |
| Performance impact on battery | Medium | Configurable timing, optimize code |
| Animation smoothness | Low | Adjust update interval if needed |
| Fork maintenance burden | Medium | Regular upstream merges |

### 8.2 Compatibility Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Breaking changes in upstream ZMK | Medium | Monitor ZMK releases, test regularly |
| Hardware compatibility issues | Low | Test thoroughly before deployment |
| Build pipeline differences | Low | Use GitHub Actions like upstream |

---

## 9. Alternative Approaches

### 9.1 Approach A: Single LED Status Indicator

**Description:** Use a single RGB LED to cycle through different status information

**Pros:**
- Simpler implementation
- No LED mapping needed
- Could work with existing ZMK modules

**Cons:**
- Less information displayed
- Sequential display only
- User must wait for desired info

**Feasibility:** HIGH (90%)

### 9.2 Approach B: Simple Status Macro (No Animation)

**Description:** Implement RGB_STATUS without fade animation, fixed 5-10 second display

**Pros:**
- Simpler code
- Less CPU usage
- Easier to maintain

**Cons:**
- Less polished appearance
- Abrupt transitions

**Feasibility:** VERY HIGH (95%)
**Recommendation:** Good starting point; add animation in Phase 3

### 9.3 Approach C: Event-Driven LED Updates

**Description:** Update specific LEDs immediately when status changes (not temporary display)

**Pros:**
- Real-time status always visible
- No timer overhead
- Simpler state management

**Cons:**
- LEDs always showing status (not normal RGB effects)
- May conflict with aesthetic preferences
- Less battery efficient

**Feasibility:** HIGH (85%)

---

## 10. Recommendations

### 10.1 Recommended Path Forward

**✅ YES - RGB Status is Feasible for Imprint**

**Recommended Implementation:**

1. **Create Custom ZMK Fork**
   - Fork zmkfirmware/zmk
   - Set up automated builds
   - Configure regular upstream merges

2. **Implement Phase 1 (Core Features)**
   - RGB_STATUS command infrastructure
   - Battery level (central only)
   - Layer status
   - Bluetooth profiles
   - USB status
   - Simple timer-based display (no animation initially)

3. **Test and Validate**
   - Verify LED count and mapping
   - Test all status displays
   - Optimize performance

4. **Implement Phase 2 (Enhanced)**
   - Add HID lock indicators
   - Investigate peripheral battery
   - Add if possible

5. **Implement Phase 3 (Polish)**
   - Add fade animations
   - Refine LED mapping
   - Add configuration options

### 10.2 Feature Priority Matrix

| Feature | Feasibility | User Value | Priority |
|---------|-------------|------------|----------|
| Battery level (central) | ✅ High | High | **P0** |
| Bluetooth profiles | ✅ Very High | High | **P0** |
| Layer status | ✅ High | Medium | **P1** |
| USB status | ✅ Very High | Medium | **P1** |
| Lock indicators | ✅ High | Low | **P2** |
| Peripheral battery | ⚠️ Medium | Medium | **P2** |
| Fade animation | ✅ High | Low | **P3** |

### 10.3 Go/No-Go Decision

**RECOMMENDATION: GO** ✅

**Justification:**
- RGB hardware confirmed present on Imprint
- 80%+ of desired APIs available in vanilla ZMK
- Similar implementation exists (Glove80) as reference
- Moderate effort with high value for users
- Clear implementation path

**Caveats:**
- Requires maintaining a custom ZMK fork
- Peripheral battery may need workaround
- LED mapping requires empirical testing
- Estimated 32-53 hours development time

---

## 11. Next Steps

If proceeding with implementation:

1. ✅ **Create ZMK Fork**
   - Fork zmkfirmware/zmk on GitHub
   - Set up build pipeline
   - Create feature branch `feature/rgb-status`

2. ✅ **Determine LED Configuration**
   - Flash test firmware to identify LED count
   - Map LED positions physically
   - Document LED chain order

3. ✅ **Implement Minimal Viable Feature**
   - Add RGB_STATUS command
   - Implement basic battery display only
   - Test on hardware

4. ✅ **Iterate and Expand**
   - Add more status displays
   - Refine LED mapping
   - Add configuration options

5. ✅ **Document and Share**
   - Create implementation guide
   - Share with Cyboard community
   - Consider contributing generalized version to ZMK

---

## 12. References

### 12.1 Documentation Reviewed

- Glove80 RGB Status Implementation Guide
- ZMK Firmware Official Documentation
- ZMK GitHub Repository Source Code
- Imprint Configuration Files

### 12.2 Key Source Files

**Vanilla ZMK:**
- `app/include/dt-bindings/zmk/rgb.h`
- `app/src/behaviors/behavior_rgb_underglow.c`
- `app/src/rgb_underglow.c`
- `app/include/zmk/ble.h`
- `app/include/zmk/keymap.h`
- `app/include/zmk/hid_indicators.h`
- `app/include/zmk/usb.h`
- `app/include/zmk/endpoints.h`

**Imprint Configuration:**
- `config/imprint.conf`
- `config/imprint.keymap`
- `build.yaml`

---

## Report Metadata

**Generated:** 2025-11-06
**Keyboard:** Cyboard Imprint
**Firmware:** ZMK (vanilla, main branch)
**Report Version:** 1.0
**Author:** Claude Code Analysis

---

## Appendix A: Feature Comparison Matrix

| Feature | Glove80 (Fork) | Vanilla ZMK | Imprint (Proposed) |
|---------|----------------|-------------|-------------------|
| RGB Underglow | ✅ | ✅ | ✅ |
| RGB_STATUS Command | ✅ | ❌ | 🔨 To Implement |
| Battery Display | ✅ | N/A | 🔨 To Implement |
| BT Profile Display | ✅ | N/A | 🔨 To Implement |
| Layer Display | ✅ | N/A | 🔨 To Implement |
| Lock Indicators | ✅ | N/A | 🔨 To Implement |
| USB Status | ✅ | N/A | 🔨 To Implement |
| Peripheral Battery | ✅ | ⚠️ Limited | ⚠️ Investigate |
| Fade Animation | ✅ | N/A | 🔨 Phase 3 |

Legend:
- ✅ Supported
- ❌ Not Available
- ⚠️ Partially Available / Uncertain
- 🔨 Planned Implementation
- N/A: Not Applicable (feature layer)

---

## Appendix B: Configuration Template

Proposed configuration additions for `config/imprint.conf`:

```conf
# RGB Status Feature Configuration
CONFIG_ZMK_RGB_STATUS=y

# Required for HID indicators (Caps Lock, etc.)
CONFIG_ZMK_HID_INDICATORS=y

# Required for peripheral battery reporting (split keyboard)
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y

# RGB Status Display Duration (milliseconds)
CONFIG_ZMK_RGB_STATUS_DURATION=10000

# RGB Status Update Interval (milliseconds)
CONFIG_ZMK_RGB_STATUS_UPDATE_INTERVAL=25

# Enable fade animation
CONFIG_ZMK_RGB_STATUS_FADE_ANIMATION=y
```

---

*End of Report*
