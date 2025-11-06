# Security Contributing Guide

This guide provides detailed information for contributors on how to maintain security when developing NeuroNexusZed.

## Table of Contents

- [Security First Mindset](#security-first-mindset)
- [Common Security Pitfalls](#common-security-pitfalls)
- [Secure Coding Practices](#secure-coding-practices)
- [Code Review Security Checklist](#code-review-security-checklist)
- [Testing Security Features](#testing-security-features)

## Security First Mindset

When contributing to NeuroNexusZed, always consider:

1. **Input Validation**: Never trust user input
2. **Error Handling**: Errors can leak sensitive information
3. **Memory Safety**: Leverage Rust's safety guarantees
4. **Dependency Security**: Keep dependencies updated and audited
5. **Principle of Least Privilege**: Grant minimum necessary permissions

## Common Security Pitfalls

### 1. Panic-Prone Code

**BAD** - Can cause denial of service:
```rust
fn process_data(data: &str) -> String {
    let parts: Vec<&str> = data.split(',').collect();
    parts[0].to_string()  // PANIC if empty!
}
```

**GOOD** - Handles errors gracefully:
```rust
fn process_data(data: &str) -> Result<String> {
    let parts: Vec<&str> = data.split(',').collect();
    parts.first()
        .ok_or_else(|| anyhow!("No data found"))
        .map(|s| s.to_string())
}
```

### 2. Silently Discarding Errors

**BAD** - Errors are hidden:
```rust
let _ = write_to_file(&data);  // Error silently ignored
```

**GOOD** - Errors are handled:
```rust
write_to_file(&data)?;  // Propagate error
// OR
write_to_file(&data).log_err();  // Log error if must ignore
```

### 3. Unsafe Code Without Documentation

**BAD** - No safety invariants documented:
```rust
unsafe {
    let ptr = data.as_ptr();
    *ptr = 0;
}
```

**GOOD** - Safety invariants clearly documented:
```rust
// SAFETY: `data` is guaranteed to be non-empty by the check above,
// and we're writing to the first element which is always valid.
unsafe {
    debug_assert!(!data.is_empty());
    let ptr = data.as_mut_ptr();
    *ptr = 0;
}
```

### 4. Path Traversal Vulnerabilities

**BAD** - Allows directory traversal:
```rust
fn read_user_file(filename: &str) -> Result<String> {
    let path = format!("./data/{}", filename);  // Can be "../../../etc/passwd"
    std::fs::read_to_string(path)
}
```

**GOOD** - Validates and sanitizes paths:
```rust
fn read_user_file(filename: &str) -> Result<String> {
    // Reject paths with directory components
    if filename.contains("..") || filename.contains('/') || filename.contains('\\') {
        return Err(anyhow!("Invalid filename"));
    }
    
    let base_path = PathBuf::from("./data");
    let path = base_path.join(filename);
    
    // Ensure canonical path is still under base_path
    let canonical = path.canonicalize()?;
    if !canonical.starts_with(base_path.canonicalize()?) {
        return Err(anyhow!("Path traversal detected"));
    }
    
    std::fs::read_to_string(canonical)
}
```

### 5. Command Injection

**BAD** - Vulnerable to command injection:
```rust
fn run_command(user_input: &str) -> Result<String> {
    let output = std::process::Command::new("sh")
        .arg("-c")
        .arg(format!("echo {}", user_input))  // DANGEROUS!
        .output()?;
    Ok(String::from_utf8_lossy(&output.stdout).to_string())
}
```

**GOOD** - Uses proper argument passing:
```rust
fn run_command(user_input: &str) -> Result<String> {
    let output = std::process::Command::new("echo")
        .arg(user_input)  // Properly escaped
        .output()?;
    Ok(String::from_utf8_lossy(&output.stdout).to_string())
}
```

## Secure Coding Practices

### Input Validation

Always validate input at trust boundaries:

```rust
fn process_port(port_str: &str) -> Result<u16> {
    let port: u16 = port_str.parse()
        .context("Invalid port number")?;
    
    // Validate range
    if port < 1024 {
        return Err(anyhow!("Port must be >= 1024"));
    }
    
    Ok(port)
}
```

### Secure Randomness

Use cryptographically secure random number generators:

```rust
use rand::Rng;

// GOOD - Cryptographically secure
fn generate_token() -> String {
    let mut rng = rand::thread_rng();
    let bytes: [u8; 32] = rng.gen();
    hex::encode(bytes)
}
```

### Credential Handling

Never log, print, or store credentials in plain text:

```rust
use zeroize::Zeroize;

#[derive(Zeroize)]
#[zeroize(drop)]
struct Credentials {
    username: String,
    password: String,
}

impl Drop for Credentials {
    fn drop(&mut self) {
        // Password is automatically zeroized
    }
}

// Don't derive Debug for credential types
impl std::fmt::Debug for Credentials {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Credentials")
            .field("username", &self.username)
            .field("password", &"[REDACTED]")
            .finish()
    }
}
```

### Integer Overflow Protection

Enable overflow checks in release builds:

```rust
// In Cargo.toml
[profile.release]
overflow-checks = true

// In code, use checked operations for critical calculations
fn calculate_buffer_size(count: usize, size: usize) -> Result<usize> {
    count.checked_mul(size)
        .ok_or_else(|| anyhow!("Buffer size overflow"))
}
```

## Code Review Security Checklist

When reviewing code, check for:

- [ ] No uses of `unwrap()`, `expect()`, or panics in error paths
- [ ] All errors are properly handled or explicitly logged
- [ ] All `unsafe` blocks are documented with safety invariants
- [ ] Input validation for all external data
- [ ] No hardcoded credentials or secrets
- [ ] Path operations check for traversal attacks
- [ ] Command execution uses proper argument escaping
- [ ] Cryptographic operations use secure algorithms
- [ ] Integer operations check for overflow
- [ ] New dependencies are reviewed for security
- [ ] Sensitive data is zeroized when no longer needed
- [ ] Debug implementations don't leak sensitive data

## Testing Security Features

### Writing Security Tests

```rust
#[cfg(test)]
mod security_tests {
    use super::*;

    #[test]
    fn test_path_traversal_protection() {
        // Should reject path traversal attempts
        assert!(read_user_file("../../../etc/passwd").is_err());
        assert!(read_user_file("..\\..\\..\\windows\\system32").is_err());
        
        // Should accept valid filenames
        assert!(read_user_file("valid_file.txt").is_ok());
    }

    #[test]
    fn test_input_validation() {
        // Should reject invalid inputs
        assert!(process_port("0").is_err());
        assert!(process_port("99999").is_err());
        assert!(process_port("abc").is_err());
        
        // Should accept valid inputs
        assert!(process_port("8080").is_ok());
    }

    #[test]
    fn test_no_panic_on_empty_input() {
        // Ensure functions handle empty input gracefully
        let result = std::panic::catch_unwind(|| {
            process_data("")
        });
        assert!(result.is_ok(), "Function panicked on empty input");
    }
}
```

### Fuzzing Critical Components

For security-critical code, consider adding fuzz tests:

```rust
#[cfg(fuzzing)]
pub mod fuzz {
    use super::*;
    
    pub fn fuzz_process_data(data: &[u8]) {
        if let Ok(s) = std::str::from_utf8(data) {
            let _ = process_data(s);  // Should never panic
        }
    }
}
```

## Additional Resources

- [Rust Security Guidelines](https://anssi-fr.github.io/rust-guide/)
- [OWASP Secure Coding Practices](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/)
- [Rust Secure Code Working Group](https://github.com/rust-secure-code)
- [Clippy Security Lints](https://rust-lang.github.io/rust-clippy/master/)

## Questions?

If you have security-related questions while contributing, please:

1. Check this guide and the main [SECURITY.md](../SECURITY.md)
2. Ask in the pull request review
3. Contact the security team (for sensitive questions)

Remember: **When in doubt, ask for a security review!**
