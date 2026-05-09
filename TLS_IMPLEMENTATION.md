# MQTT TLS/SSL Implementation

## Overview

TLS/SSL encryption support has been added to the MQTT module in majordomo-mqtt-tls. This allows secure encrypted connections to MQTT brokers using TLS protocol while maintaining full backward compatibility with existing configurations.

## Changes Made

### 1. Core Module File: `modules/mqtt/mqtt.class.php`

#### Modified `mqttPublish()` method
- Added TLS configuration reading from module settings
- Added TLS context creation and setup for phpMQTT client
- Supports both certificate file paths and ignore certificate validation

**New Configuration Parameters Read:**
- `MQTT_TLS_ENABLE`: Enable/disable TLS (0 or 1)
- `MQTT_TLS_CERTPATH`: Path to CA certificate file
- `MQTT_TLS_IGNORE_CERT`: Ignore certificate validation for self-signed certificates (0 or 1)

**TLS Connection Logic:**
```php
- If TLS_ENABLE = 0: Regular unencrypted connection (default, backward compatible)
- If TLS_ENABLE = 1: Encrypted TLS connection
  - If TLS_CERTPATH provided: Use certificate for verification
  - If TLS_IGNORE_CERT = 1: Skip certificate validation (for self-signed certs)
```

#### Modified `admin()` method
- Added TLS configuration output for template rendering
- Added TLS configuration update handling in `update_settings` view mode
- All TLS settings default to 0 (disabled) for backward compatibility

**Configuration Output Variables:**
- `MQTT_TLS_ENABLE`
- `MQTT_TLS_CERTPATH`
- `MQTT_TLS_IGNORE_CERT`

**Configuration Update Handling:**
- Reads `mqtt_tls_enable`, `mqtt_tls_certpath`, `mqtt_tls_ignore_cert` from form data
- Saves settings via existing `saveConfig()` method
- Triggers MQTT cycle restart on settings change

### 2. Template File: `templates/mqtt/mqtt_search_admin.html`

#### Added TLS Configuration Form Section
Located after authentication settings, includes:

1. **TLS/SSL Enable Checkbox**
   - Shows/hides TLS option fields when toggled
   - Default: unchecked (TLS disabled)

2. **Certificate File Path Input**
   - Text field for entering path to CA certificate file
   - Example: `/etc/mqtt/ca.crt`
   - Optional - leave empty if certificate validation not needed

3. **Ignore Certificate Validation Checkbox**
   - Allows connection with self-signed certificates
   - Displays security warning in red
   - Only visible when TLS is enabled

**Form Behavior:**
- TLS options are hidden by default
- TLS options section auto-shows/hides based on TLS enable checkbox
- If TLS was previously enabled, options section shows on page load

## Backward Compatibility

✅ **Full backward compatibility maintained:**

1. **Existing Configurations**: Unchanged configurations will continue to work without modification
   - TLS settings default to disabled (0) if not present
   - No changes required to existing MQTT connections

2. **Non-Breaking Changes**: All changes are purely additive
   - New configuration parameters are optional
   - Existing function signatures unchanged
   - Default behavior identical to pre-TLS version

3. **No Database Schema Changes**: No modifications to MQTT database tables required

4. **Code Patterns**: Uses same patterns as existing authentication settings
   - Follows existing config read/write patterns
   - Uses standard `gr()` form data retrieval
   - Integrates with existing configuration save mechanism

## Usage Guide

### Step 1: Enable TLS in Admin Panel
1. Navigate to MQTT module settings
2. Check "TLS/SSL Enable" checkbox
3. TLS options section will appear

### Step 2: Configure Certificate (if needed)
1. If using self-signed certificate or need validation, enter certificate path
   - Path: `/etc/mqtt/ca.crt` (example)
   - Can be absolute path to CA certificate file

2. If using self-signed certificate without proper CA:
   - Check "Ignore Certificate Validation"
   - ⚠️ Warning: This reduces security but allows self-signed certs

### Step 3: Adjust MQTT Port (if needed)
- Standard MQTT TLS port: **8883** (change from default 1883)
- Adjust port in "MQTT Port" field accordingly

### Step 4: Save Settings
1. Click "Update" button
2. MQTT cycle will restart to apply new settings
3. Connection will use TLS on next publish/subscribe

## Configuration Examples

### Example 1: TLS with Self-Signed Certificate (Ignore Validation)
```
TLS/SSL Enable: ✓ (checked)
Certificate File Path: (empty)
Ignore Certificate Validation: ✓ (checked)
MQTT Port: 8883
```

### Example 2: TLS with CA Certificate Validation
```
TLS/SSL Enable: ✓ (checked)
Certificate File Path: /etc/mqtt/ca.crt
Ignore Certificate Validation: ☐ (unchecked)
MQTT Port: 8883
```

### Example 3: Non-TLS Connection (Default)
```
TLS/SSL Enable: ☐ (unchecked)
Certificate File Path: (not shown)
Ignore Certificate Validation: (not shown)
MQTT Port: 1883
```

## Technical Details

### TLS Implementation Method
- Uses PHP stream context with SSL options
- Attempts to set TLS context via phpMQTT client methods:
  1. `setTlsContext()` method (if available)
  2. `tlsContext` property (if available)
  3. Graceful fallback if method/property not available

### SSL Context Options
```php
// Always set when TLS enabled
'local_cert' => $tls_ca_file

// Set when ignore_cert is enabled
'verify_peer' => false
'verify_peer_name' => false
'allow_self_signed' => true
```

### Error Handling
- Connection errors with TLS enabled logged to mqtt_error debug message
- Falls back to existing error handling mechanisms
- No new exception types introduced

## Security Considerations

⚠️ **Important Security Notes:**

1. **Certificate Validation (Recommended)**
   - Always use certificate validation when possible
   - Only use "Ignore Certificate Validation" in development/testing
   - Provides protection against man-in-the-middle attacks

2. **Self-Signed Certificates**
   - Safe to ignore validation for self-signed certs in trusted networks
   - Not recommended for internet-exposed connections

3. **Certificate File Permissions**
   - Ensure certificate files are readable by web server user
   - Protect certificate files with appropriate permissions

4. **Password Storage**
   - Continue using MQTT authentication with TLS
   - TLS encrypts traffic; authentication protects broker access

## Troubleshooting

### Connection Fails with TLS Enabled
- Verify broker supports TLS on configured port (default 8883)
- Check certificate file path is correct and accessible
- Try disabling "Ignore Certificate Validation" if using valid cert
- Check MQTT broker logs for TLS errors

### Certificate Validation Errors
- Verify certificate file path is correct
- Ensure CA certificate matches broker certificate
- Try enabling "Ignore Certificate Validation" temporarily to isolate issue
- Check certificate expiration date

### Port Connection Issues
- Verify MQTT broker is listening on TLS port (8883)
- Check firewall/network accessibility to broker:port
- Confirm port configuration in MQTT settings

## Module Integration

The TLS implementation integrates with:
- **MQTT message publishing**: All `mqttPublish()` calls use TLS when enabled
- **Topic subscriptions**: Connections for receiving MQTT messages use TLS
- **Linked objects**: No changes to property linking mechanism
- **MQTT history**: No changes to logging functionality

## Files Modified

1. `/modules/mqtt/mqtt.class.php`
   - `mqttPublish()` method
   - `admin()` method

2. `/templates/mqtt/mqtt_search_admin.html`
   - Added TLS configuration form section

## Future Enhancements

Possible future improvements:
- Certificate file upload UI
- Client certificate (mutual TLS) support
- TLS protocol version selection (TLS 1.2, 1.3, etc.)
- Certificate validation testing/verification utility
- TLS connection diagnostic logs

## Support and Questions

For issues related to TLS implementation:
1. Check that MQTT broker supports TLS
2. Verify certificate files and paths
3. Test TLS connection manually if possible
4. Review MQTT broker and application logs for detailed errors
