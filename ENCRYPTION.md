# Token Encryption Documentation

## Overview

All Strava OAuth tokens (access tokens and refresh tokens) are encrypted before being stored in the database using AES-256-GCM encryption. This ensures that even if the database is compromised, the tokens cannot be used without the encryption key.

## Encryption Method

- **Algorithm**: AES-256-GCM (Galois/Counter Mode)
- **Key Size**: 256 bits (32 bytes)
- **Features**:
  - Authenticated encryption (prevents tampering)
  - Unique nonce for each encryption operation
  - Base64 encoding for database storage

## Setup

### 1. Generate Encryption Key

Run the provided script to generate a secure encryption key:

```bash
cd backend
./scripts/generate-encryption-key.sh
```

This will output something like:
```
ENCRYPTION_KEY=abcd1234efgh5678ijkl9012mnop3456qrst7890uvwx==
```

### 2. Configure Environment

Add the encryption key to your `.env` file:

```bash
ENCRYPTION_KEY=your_generated_key_here
```

**IMPORTANT SECURITY NOTES:**
- ⚠️ Never commit the `.env` file to version control
- ⚠️ Keep the encryption key secure - treat it like a password
- ⚠️ If the key is lost, encrypted tokens cannot be recovered
- ⚠️ Use different keys for development and production
- ⚠️ Rotate keys periodically (requires re-encrypting all tokens)

### 3. Key Requirements

- Must be exactly 32 bytes (256 bits)
- Should be cryptographically random
- Use `openssl rand -base64 32` to generate

## Implementation Details

### Encryption Flow

1. **Token Receipt** (OAuth callback):
   ```
   Strava API → Plain Token → Encrypt → Store Encrypted Token in DB
   ```

2. **Token Usage** (API calls):
   ```
   DB → Retrieve Encrypted Token → Decrypt → Use Plain Token → Strava API
   ```

3. **Token Refresh**:
   ```
   DB → Decrypt Old Tokens → Refresh via Strava API → Encrypt New Tokens → Store in DB
   ```

### Code Locations

#### Encryption/Decryption Library
- **File**: `internal/crypto/encryption.go`
- **Functions**:
  - `Encrypt(plaintext, key string) (string, error)`
  - `Decrypt(ciphertext, key string) (string, error)`

#### Token Storage (Encryption)
- **File**: `internal/handlers/strava_handlers.go`
- **Function**: `ConnectStrava()`
- **Process**:
  1. Receive OAuth tokens from Strava
  2. Encrypt access token
  3. Encrypt refresh token
  4. Store encrypted tokens in database

#### Token Usage (Decryption)
- **File**: `internal/services/strava_service.go`
- **Function**: `ensureValidToken()`
- **Process**:
  1. Retrieve encrypted tokens from database
  2. Decrypt access token
  3. Check expiration
  4. If expired, decrypt refresh token and get new tokens
  5. Encrypt and store new tokens

## Security Considerations

### What is Protected
✅ Access tokens are encrypted at rest
✅ Refresh tokens are encrypted at rest
✅ Tokens are only decrypted in memory when needed
✅ Encryption is authenticated (prevents tampering)

### What is NOT Protected
❌ Tokens in memory during processing
❌ Tokens in transit (use HTTPS for this)
❌ Tokens in application logs (never log tokens!)

### Best Practices

1. **Environment Security**:
   - Use secure environment variable management (e.g., AWS Secrets Manager, HashiCorp Vault)
   - Restrict access to environment variables
   - Use different keys per environment

2. **Key Management**:
   - Store keys separately from the application
   - Use key rotation policies
   - Have a backup/recovery plan
   - Consider using a Key Management Service (KMS)

3. **Database Security**:
   - Even with encryption, secure your database
   - Use database encryption at rest
   - Restrict database access
   - Monitor for unusual access patterns

4. **Application Security**:
   - Never log decrypted tokens
   - Clear tokens from memory when done
   - Use HTTPS for all communications
   - Implement rate limiting

## Key Rotation

To rotate the encryption key:

1. **Generate new key**:
   ```bash
   ./scripts/generate-encryption-key.sh
   ```

2. **Decrypt with old key, re-encrypt with new key**:
   ```go
   // Pseudocode for migration
   oldKey := os.Getenv("OLD_ENCRYPTION_KEY")
   newKey := os.Getenv("NEW_ENCRYPTION_KEY")
   
   for each connection in database {
       accessToken = crypto.Decrypt(connection.AccessToken, oldKey)
       refreshToken = crypto.Decrypt(connection.RefreshToken, oldKey)
       
       connection.AccessToken = crypto.Encrypt(accessToken, newKey)
       connection.RefreshToken = crypto.Encrypt(refreshToken, newKey)
       
       update database
   }
   ```

3. **Update environment variable**:
   ```bash
   ENCRYPTION_KEY=new_key_here
   ```

4. **Restart application**

## Testing

### Test Encryption/Decryption

```go
package crypto

import "testing"

func TestEncryptDecrypt(t *testing.T) {
    key := "12345678901234567890123456789012" // 32 bytes
    plaintext := "strava_access_token_12345"
    
    encrypted, err := Encrypt(plaintext, key)
    if err != nil {
        t.Fatalf("Encryption failed: %v", err)
    }
    
    decrypted, err := Decrypt(encrypted, key)
    if err != nil {
        t.Fatalf("Decryption failed: %v", err)
    }
    
    if decrypted != plaintext {
        t.Fatalf("Expected %s, got %s", plaintext, decrypted)
    }
}
```

### Verify Token Storage

```bash
# Connect Strava account via UI
# Check database - tokens should be base64-encoded encrypted strings
sqlite3 database.db "SELECT access_token FROM strava_connections LIMIT 1;"
# Should show something like: "SGVsbG8gV29ybGQh..." (not a plain token)
```

## Troubleshooting

### Error: "encryption key must be 32 bytes"
- Your `ENCRYPTION_KEY` is not exactly 32 bytes
- Generate a new key using the provided script

### Error: "ENCRYPTION_KEY not set"
- The environment variable is missing
- Add it to your `.env` file

### Error: "failed to decrypt"
- Wrong encryption key
- Token was encrypted with a different key
- Database corruption
- Need to re-authenticate with Strava

### Tokens not working after key change
- Old tokens were encrypted with the old key
- Either:
  1. Decrypt with old key and re-encrypt with new key (migration)
  2. Ask users to reconnect their Strava accounts

## Compliance

This encryption implementation helps with:
- **GDPR**: Protecting personal data (OAuth tokens)
- **PCI DSS**: Protecting sensitive authentication data
- **SOC 2**: Security controls for data protection
- **HIPAA**: If handling health data alongside authentication

## Performance Impact

- **Encryption**: ~1-2ms per token
- **Decryption**: ~1-2ms per token
- **Impact**: Negligible for most operations
- **Caching**: Decrypted tokens are used in memory during request processing

## Future Enhancements

Consider implementing:
- [ ] Hardware Security Module (HSM) integration
- [ ] Key rotation automation
- [ ] Token expiration tracking
- [ ] Audit logging for decryption operations
- [ ] Multiple encryption keys with key versioning
- [ ] Integration with cloud KMS (AWS KMS, Google Cloud KMS)
