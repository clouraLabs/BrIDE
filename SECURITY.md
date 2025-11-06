# Security Policy

## Supported Versions

We actively support security updates for the following versions:

| Version | Supported          |
| ------- | ------------------ |
| main    | :white_check_mark: |
| latest  | :white_check_mark: |
| < 1.0   | :x:                |

## Reporting a Vulnerability

We take the security of NeuroNexusZed seriously. If you believe you have found a security vulnerability, please report it to us as described below.

### Please do NOT:
- Open a public GitHub issue for security vulnerabilities
- Discuss potential vulnerabilities in public forums or social media
- Attempt to exploit the vulnerability beyond what is necessary to demonstrate it

### Please DO:
1. **Report via GitHub Security Advisories** (Preferred)
   - Navigate to the Security tab of this repository
   - Click "Report a vulnerability"
   - Provide detailed information about the vulnerability

2. **Email Report** (Alternative)
   - Send details to: [security@clouralabs.com] (update with your actual security contact)
   - Use PGP encryption if possible

### What to Include in Your Report

To help us better understand and resolve the issue, please include:

- **Type of vulnerability** (e.g., XSS, SQL injection, unsafe Rust code, etc.)
- **Full path and line numbers** of source file(s) related to the vulnerability
- **Location of the affected code** (tag/branch/commit or direct URL)
- **Step-by-step instructions** to reproduce the issue
- **Proof-of-concept or exploit code** (if possible)
- **Impact assessment** - what an attacker could do with this vulnerability
- **Suggested fix** (if you have one)

## Response Timeline

- **Initial Response**: Within 48 hours of receiving your report
- **Status Update**: Weekly updates on progress
- **Fix Timeline**: We aim to patch critical vulnerabilities within 7 days
- **Disclosure**: Coordinated disclosure after patch is available

## Security Best Practices

### For Contributors

When contributing to NeuroNexusZed, please follow these security guidelines:

#### 1. Unsafe Rust Code
- Minimize use of `unsafe` blocks
- Document all `unsafe` code with clear safety invariants
- Have `unsafe` code reviewed by multiple maintainers
- Ensure all safety conditions are met before using unsafe operations

#### 2. Error Handling
- **NEVER use `unwrap()` or `expect()` on operations that can fail**
- Use `?` to propagate errors appropriately
- Use `.log_err()` when errors must be ignored but should be logged
- Avoid `let _ =` on fallible operations

Example - Bad:
```rust
let _ = client.request(...).await?;  // Silently discards error
let value = option.unwrap();  // Can panic
```

Example - Good:
```rust
client.request(...).await?;  // Propagates error
let value = option.ok_or_else(|| anyhow!("Missing value"))?;
```

#### 3. Input Validation
- Validate all user inputs at boundaries
- Sanitize data before use in system calls or SQL
- Use parameterized queries for database operations
- Validate file paths to prevent directory traversal

#### 4. Dependencies
- Keep dependencies up to date
- Review security advisories for dependencies
- Minimize dependency count where practical
- Prefer well-maintained, audited crates

#### 5. Credentials and Secrets
- NEVER commit secrets, API keys, or credentials
- Use environment variables or secure credential storage
- Rotate credentials if accidentally exposed
- Use `.gitignore` to exclude sensitive files

### For Users

#### Running from Source
- Always verify the source code before building
- Use official releases when possible
- Check GPG signatures on releases
- Keep your build toolchain updated

#### Reporting Issues
- Report crashes that might indicate security issues
- Share logs (after redacting sensitive information)
- Update to latest version before reporting

## Security Features

### Current Security Measures

1. **Memory Safety**: Written in Rust for memory safety guarantees
2. **Dependency Management**: Regular dependency audits
3. **Code Review**: All PRs require review before merging
4. **Static Analysis**: Clippy and other linters in CI/CD
5. **Sandboxing**: Extension system uses WebAssembly sandboxing

### Planned Security Enhancements

- [ ] Automated security scanning in CI/CD
- [ ] Regular third-party security audits
- [ ] Fuzzing of critical components
- [ ] Security-focused unit tests
- [ ] Runtime security monitoring

## Security-Related Configuration

### Recommended Rust Compiler Flags

```toml
[profile.release]
overflow-checks = true  # Catch integer overflows in release builds
```

### Clippy Security Lints

We enforce these security-related lints:

- `deny(clippy::unwrap_used)` - Prevent panics from unwrap
- `deny(clippy::expect_used)` - Prevent panics from expect
- `deny(clippy::panic)` - Prevent explicit panics
- `deny(clippy::mem_forget)` - Prevent memory leaks
- `warn(clippy::todo)` - Track incomplete code

## Known Security Considerations

### Unsafe Code Locations

The codebase contains `unsafe` code in the following areas:
- Platform-specific FFI bindings (Windows, macOS, Linux)
- SQLite C API interactions (`crates/sqlez`)
- Low-level memory operations in performance-critical paths
- Terminal emulator implementation
- Graphics and rendering systems

All unsafe code has been reviewed and documented with safety invariants.

### Third-Party Dependencies

We regularly audit our dependencies for security issues. Major dependencies include:
- `tokio` - Async runtime
- `reqwest` - HTTP client (using our patched fork with security fixes)
- `rustls` - TLS implementation
- `wasmtime` - WebAssembly runtime for extensions

## Security Hall of Fame

We recognize and thank security researchers who have helped improve NeuroNexusZed's security:

<!-- Add contributors here as vulnerabilities are responsibly disclosed -->
- Your name could be here!

## Additional Resources

- [Rust Security Guidelines](https://anssi-fr.github.io/rust-guide/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Secure Coding in Rust](https://docs.rust-embedded.org/book/static-guarantees/index.html)

## Legal

This security policy is subject to our [Terms of Service](legal/terms.md) and [Privacy Policy](legal/privacy-policy.md).

## Updates to This Policy

This security policy may be updated from time to time. Material changes will be announced in release notes.

**Last Updated**: 2025-11-06
