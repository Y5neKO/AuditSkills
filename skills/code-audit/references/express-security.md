# Express.js Security Patterns Reference

## Common Vulnerability Patterns

### 1. SQL Injection

**Vulnerable Pattern:**
```javascript
app.get('/users/:id', (req, res) => {
  const userId = req.params.id;
  db.query(`SELECT * FROM users WHERE id = ${userId}`, (err, results) => {
    res.json(results);
  });
});
```

**Safe Pattern:**
```javascript
app.get('/users/:id', (req, res) => {
  const userId = req.params.id;
  db.query('SELECT * FROM users WHERE id = ?', [userId], (err, results) => {
    res.json(results);
  });
});
```

### 2. XSS (Cross-Site Scripting)

**Vulnerable Pattern:**
```javascript
app.get('/search', (req, res) => {
  const query = req.query.q;
  res.send(`<h1>Results for: ${query}</h1>`);
});
```

**Safe Pattern:**
```javascript
const he = require('he');

app.get('/search', (req, res) => {
  const query = he.encode(req.query.q);
  res.send(`<h1>Results for: ${query}</h1>`);
});
```

### 3. Command Injection

**Vulnerable Pattern:**
```javascript
app.post('/backup', (req, res) => {
  const filename = req.body.filename;
  exec(`tar -czf backup_${filename}.tar.gz /data`, (err, stdout) => {
    res.send('Backup created');
  });
});
```

**Safe Pattern:**
```javascript
const { spawn } = require('child_process');

app.post('/backup', (req, res) => {
  const filename = req.body.filename;
  // Validate filename
  if (!/^[a-zA-Z0-9_-]+$/.test(filename)) {
    return res.status(400).send('Invalid filename');
  }
  
  const tar = spawn('tar', ['-czf', `backup_${filename}.tar.gz`, '/data']);
  tar.on('close', (code) => {
    res.send('Backup created');
  });
});
```

### 4. Path Traversal

**Vulnerable Pattern:**
```javascript
app.get('/download', (req, res) => {
  const file = req.query.file;
  res.sendFile(`/uploads/${file}`);
});
```

**Safe Pattern:**
```javascript
const path = require('path');

app.get('/download', (req, res) => {
  const file = req.query.file;
  const safePath = path.normalize(`/uploads/${file}`).replace(/^(\.\.(\/|\\|$))+/, '');
  
  if (!safePath.startsWith('/uploads/')) {
    return res.status(400).send('Invalid path');
  }
  
  res.sendFile(safePath);
});
```

## Express.js Security Middleware

### Helmet.js
Protects against various web vulnerabilities by setting HTTP headers.

```javascript
const helmet = require('helmet');
app.use(helmet());
```

### CSRF Protection
```javascript
const csrf = require('csurf');
const csrfProtection = csrf({ cookie: true });

app.use(csrfProtection);
```

### Rate Limiting
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/api/', limiter);
```

## Common Dangerous Functions

| Function | Risk | Alternative |
|----------|------|-------------|
| `eval()` | Code injection | Parse JSON properly |
| `Function()` | Code injection | Use proper code structure |
| `exec()` | Command injection | Use `spawn()` with args array |
| Template literal in SQL | SQL injection | Parameterized queries |
| `res.send(userInput)` | XSS | Use template engine with auto-escape |

## Global Middleware Patterns

**Authentication Check:**
```javascript
function requireAuth(req, res, next) {
  if (!req.session.userId) {
    return res.status(401).send('Unauthorized');
  }
  next();
}

app.use('/api/protected/*', requireAuth);
```

**Input Sanitization:**
```javascript
const mongoSanitize = require('express-mongo-sanitize');
app.use(mongoSanitize());

const xss = require('xss-clean');
app.use(xss());
```
