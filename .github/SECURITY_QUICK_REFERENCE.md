# Security Quick Reference

A quick reference guide for common security practices in NeuroNexusZed development.

## ⚠️ Never Do This

```rust
// ❌ DON'T: Use unwrap() - can panic
let value = option.unwrap();

// ❌ DON'T: Silently discard errors
let _ = dangerous_operation();

// ❌ DON'T: Use unsafe without documentation
unsafe { *ptr = value; }

// ❌ DON'T: Concatenate user input in commands
Command::new("sh").arg("-c").arg(format!("rm {}", user_file));

// ❌ DON'T: Trust user-provided paths
let path = format!("./data/{}", user_filename);

// ❌ DON'T: Log sensitive data
log::info!("Password: {}", password);
```

## ✅ Always Do This

```rust
// ✅ DO: Handle errors properly
let value = option.ok_or_else(|| anyhow!("Missing value"))?;

// ✅ DO: Propagate or log errors
dangerous_operation()?;
// OR
dangerous_operation().log_err();

// ✅ DO: Document safety invariants
// SAFETY: ptr is guaranteed valid by the check above
unsafe { *ptr = value; }

// ✅ DO: Use proper argument passing
Command::new("rm").arg(user_file);

// ✅ DO: Validate and sanitize paths
if user_filename.contains("..") {
    return Err(anyhow!("Invalid path"));
}

// ✅ DO: Redact sensitive data
log::info!("User authenticated: {}", username);
```

## Common Patterns

### Safe Error Handling
```rust
// Pattern 1: Propagate errors
fn operation() -> Result<T> {
    let result = fallible_call()?;
    Ok(result)
}

// Pattern 2: Provide context
fn operation() -> Result<T> {
    let result = fallible_call()
        .context("Failed to perform operation")?;
    Ok(result)
}

// Pattern 3: Log and continue
fn operation() {
    if let Err(e) = fallible_call() {
        log::error!("Operation failed: {:?}", e);
    }
}
```

### Safe Path Handling
```rust
fn safe_path(user_input: &str) -> Result<PathBuf> {
    // Reject directory traversal
    if user_input.contains("..") || user_input.contains('/') {
        return Err(anyhow!("Invalid path"));
    }
    
    let base = PathBuf::from("./safe_dir");
    let path = base.join(user_input);
    
    // Verify canonical path is under base
    let canonical = path.canonicalize()?;
    if !canonical.starts_with(base.canonicalize()?) {
        return Err(anyhow!("Path outside safe directory"));
    }
    
    Ok(canonical)
}
```

### Safe Command Execution
```rust
fn safe_command(user_arg: &str) -> Result<Output> {
    // Use smol::process for async compatibility
    let output = smol::process::Command::new("safe-command")
        .arg(user_arg)  // Args are automatically escaped
        .output()
        .await?;
    Ok(output)
}
```

### Handling Credentials
```rust
use zeroize::Zeroize;

#[derive(Zeroize)]
#[zeroize(drop)]
struct ApiKey(String);

impl std::fmt::Debug for ApiKey {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "ApiKey([REDACTED])")
    }
}
```

## Pre-Commit Checklist

Before committing code with security implications:

- [ ] No `unwrap()` or `expect()` in production code
- [ ] All errors are handled or logged
- [ ] All `unsafe` blocks are documented
- [ ] Input validation for user data
- [ ] No hardcoded secrets
- [ ] Tests cover error cases
- [ ] Security implications documented

## Running Security Checks Locally

```bash
# Run clippy with security lints
./script/clippy

# Check for known vulnerabilities (after installing cargo-audit)
cargo audit

# Check for banned/vulnerable dependencies (after installing cargo-deny)
cargo deny check

# Count unsafe blocks
rg -c "unsafe" --type rust
```

## Getting Help

- See [SECURITY.md](../../SECURITY.md) for the full security policy
- See [SECURITY_CONTRIBUTING.md](./SECURITY_CONTRIBUTING.md) for detailed guidelines
- Ask in PR reviews for security-sensitive code
- Contact security team for private security concerns

## Additional Resources

- [Rust Security Guidelines](https://anssi-fr.github.io/rust-guide/)
- [Clippy Lints](https://rust-lang.github.io/rust-clippy/master/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
