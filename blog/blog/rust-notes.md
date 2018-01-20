is_draft: true
---
```rust
let opt = match opt {
    Some(r) => {
        let val = r?;
        Some(val)
    }
    None => None,
};
```

```rust
let opt = opt.map_or(Ok(None), |r| r.map(Some))?;
```
