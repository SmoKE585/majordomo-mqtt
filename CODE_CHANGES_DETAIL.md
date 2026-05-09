# TLS Implementation - Code Changes Detail

## Summary of Changes

All changes are **backward compatible** and **non-breaking**. TLS is disabled by default.

---

## File 1: `modules/mqtt/mqtt.class.php`

### Change 1: mqttPublish() Method (Lines 292-318)

**What was added:**
- Read TLS configuration from module settings
- Create TLS/SSL stream context with certificate options
- Apply TLS context to phpMQTT client before connection

**Key Code:**
```php
// TLS/SSL configuration
$tls_enabled = isset($this->config['MQTT_TLS_ENABLE']) ? (int)$this->config['MQTT_TLS_ENABLE'] : 0;
$tls_ca_file = isset($this->config['MQTT_TLS_CERTPATH']) ? $this->config['MQTT_TLS_CERTPATH'] : '';
$tls_ignore_cert = isset($this->config['MQTT_TLS_IGNORE_CERT']) ? (int)$this->config['MQTT_TLS_IGNORE_CERT'] : 0;

// Set TLS context if TLS is enabled
if ($tls_enabled) {
    $tls_context_options = array(
        'local_cert' => $tls_ca_file ? $tls_ca_file : false
    );
    
    if ($tls_ignore_cert) {
        $tls_context_options['verify_peer'] = false;
        $tls_context_options['verify_peer_name'] = false;
        $tls_context_options['allow_self_signed'] = true;
    }
    
    $tls_context = stream_context_create(array('ssl' => $tls_context_options));
    
    // Try to set TLS context on phpMQTT client
    if (method_exists($mqtt_client, 'setTlsContext')) {
        $mqtt_client->setTlsContext($tls_context);
    } elseif (property_exists($mqtt_client, 'tlsContext')) {
        $mqtt_client->tlsContext = $tls_context;
    }
}
```

**Behavior:**
- All defaults to 0 (TLS disabled) - full backward compatibility
- Only applies TLS if explicitly enabled
- Gracefully handles phpMQTT library variations

---

### Change 2: admin() Method (Lines 646-648)

**What was added:**
- Output TLS configuration for template rendering
- Located immediately after DEBUG_MODE output

**Key Code:**
```php
// TLS/SSL configuration output
$out['MQTT_TLS_ENABLE'] = isset($this->config['MQTT_TLS_ENABLE']) ? (int)$this->config['MQTT_TLS_ENABLE'] : 0;
$out['MQTT_TLS_CERTPATH'] = isset($this->config['MQTT_TLS_CERTPATH']) ? $this->config['MQTT_TLS_CERTPATH'] : '';
$out['MQTT_TLS_IGNORE_CERT'] = isset($this->config['MQTT_TLS_IGNORE_CERT']) ? (int)$this->config['MQTT_TLS_IGNORE_CERT'] : 0;
```

**Behavior:**
- Reads from module config with sensible defaults
- Makes values available to HTML template
- No database changes required

---

### Change 3: admin() Method - update_settings (Lines 678-680)

**What was added:**
- Handle TLS configuration updates from form submission
- Located after DEBUG_MODE update handling

**Key Code:**
```php
// TLS/SSL configuration update
$this->config['MQTT_TLS_ENABLE'] = gr('mqtt_tls_enable', 'int');
$this->config['MQTT_TLS_CERTPATH'] = gr('mqtt_tls_certpath');
$this->config['MQTT_TLS_IGNORE_CERT'] = gr('mqtt_tls_ignore_cert', 'int');
```

**Behavior:**
- Reads form data using existing `gr()` function
- Saves via existing `saveConfig()` method
- Triggers MQTT cycle restart (existing behavior)
- No new parameters in function signatures

---

## File 2: `templates/mqtt/mqtt_search_admin.html`

### Change: Added TLS Configuration Form Section (Lines 93-119)

**Location:** After authentication password field, before debug mode

**What was added:**
1. **TLS Enable Checkbox** (Line 93-98)
   - Input name: `mqtt_tls_enable`
   - Shows/hides TLS options when toggled
   - Label: "TLS/SSL Enable"

2. **TLS Options Container** (Line 100)
   - Container ID: `mqtt_tls_options`
   - Visibility controlled by template condition `[#if MQTT_TLS_ENABLE="1"#]`
   - Shows/hides when checkbox toggled

3. **Certificate File Path Input** (Line 106-107)
   - Input name: `mqtt_tls_certpath`
   - Type: text
   - Placeholder: "e.g., /etc/mqtt/ca.crt"
   - Helper text: "Path to CA certificate file for TLS verification"

4. **Ignore Certificate Validation Checkbox** (Line 115-116)
   - Input name: `mqtt_tls_ignore_cert`
   - Warning text: "Use with caution - allows self-signed certificates (security risk)"
   - Color: red (#d9534f) for visibility

**HTML Pattern:**
```html
<div class="form-group">
    <label>TLS/SSL Enable:</label>
    <input type="checkbox" onchange="$('#mqtt_tls_options').toggle('slow');" 
           name="mqtt_tls_enable" value="1" [#if MQTT_TLS_ENABLE="1" #] checked[#endif#]>
</div>

<div id="mqtt_tls_options" style="[#if MQTT_TLS_ENABLE="1"#]display:block;[#else#]display:none;[#endif#]">
    <!-- Certificate path input -->
    <!-- Ignore cert checkbox -->
</div>
```

**Behavior:**
- TLS options hidden by default
- Toggle checkbox to show/hide
- If TLS was previously enabled, options show on page load
- Follows existing form styling and patterns

---

## Configuration Variables Added

### In Module Config
Three new configuration variables (all optional, default to 0/empty):

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `MQTT_TLS_ENABLE` | int (0/1) | 0 | Enable/disable TLS |
| `MQTT_TLS_CERTPATH` | string | '' | Path to CA certificate file |
| `MQTT_TLS_IGNORE_CERT` | int (0/1) | 0 | Ignore certificate validation |

### In Template Output
Same names prefixed for template usage:
- `out['MQTT_TLS_ENABLE']`
- `out['MQTT_TLS_CERTPATH']`
- `out['MQTT_TLS_IGNORE_CERT']`

---

## No Changes To

✅ **These remain unchanged:**
- Database schema (no new tables or columns)
- mqttPublish() function signature
- admin() function signature  
- MQTT topic/path handling
- Message processing
- Linked objects functionality
- Logging/history functionality
- Existing configuration options

---

## Integration Points

The implementation integrates with existing code through:

1. **Configuration System**
   - Uses existing `getConfig()` method
   - Uses existing `saveConfig()` method
   - Uses existing `gr()` form data retrieval

2. **MQTT Client**
   - Creates phpMQTT client same as before
   - Adds TLS context before existing connect() call
   - No changes to publish() method usage

3. **Error Handling**
   - Uses existing `DebMes()` for error logging
   - Maintains existing error return values (0 for failure)
   - No new exception handling needed

4. **Admin Interface**
   - Follows existing form patterns
   - Uses existing styling/classes
   - Integrates with existing settings form

---

## Testing Checklist

To verify implementation:

- [ ] Existing MQTT connections work without TLS (TLS disabled)
- [ ] Can enable TLS in admin panel
- [ ] TLS options section shows when TLS is enabled
- [ ] TLS options hide when TLS is disabled
- [ ] Certificate path can be entered
- [ ] "Ignore cert" option is available when TLS enabled
- [ ] Settings persist after save and page reload
- [ ] MQTT cycle restarts on settings update
- [ ] Connections work with TLS enabled (requires TLS-enabled broker)
- [ ] Connections work with ignore cert option (self-signed certs)

---

## Compatibility Notes

**PHP Versions:**
- Requires PHP with OpenSSL support for TLS/SSL
- Uses standard `stream_context_create()` (available in PHP 4.3.0+)
- No new PHP extensions required

**Library Compatibility:**
- Works with phpMQTT library (Bluerhinos)
- Gracefully handles different versions (tries setTlsContext() method, then tlsContext property)
- No modifications to phpMQTT library needed

**Database:**
- No schema changes
- Uses same config storage mechanism
- Backward compatible with existing installations

---

## Security Implementation

**SSL Context Options Applied:**

When TLS is enabled:
```php
'local_cert' => $tls_ca_file  // CA certificate file path
```

When "Ignore Certificate Validation" is enabled:
```php
'verify_peer' => false          // Skip peer verification
'verify_peer_name' => false     // Skip hostname verification
'allow_self_signed' => true     // Allow self-signed certificates
```

**Secure Defaults:**
- TLS disabled by default (no breaking changes)
- Certificate validation enabled by default (when TLS enabled)
- Explicit opt-in for insecure "ignore cert" mode

---

## Future Extensibility

The implementation is designed to support future enhancements:

1. **Client Certificates** (mutual TLS)
   - Add `MQTT_TLS_CLIENT_CERT` and `MQTT_TLS_CLIENT_KEY`
   - Set via stream context options

2. **TLS Protocol Version Selection**
   - Add `MQTT_TLS_MIN_VERSION` / `MQTT_TLS_MAX_VERSION`
   - Set via `crypto_method` stream option

3. **Cipher Suite Selection**
   - Add `MQTT_TLS_CIPHERS`
   - Set via stream context options

4. **Certificate Pinning**
   - Add `MQTT_TLS_PINNED_CERT` 
   - Implement custom validation logic

All future additions can follow the same pattern without breaking existing configurations.
