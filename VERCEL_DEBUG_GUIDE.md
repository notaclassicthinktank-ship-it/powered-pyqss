# Powered PYQs - Vercel 404 Debugging Guide

## 🔍 Analysis Summary

Your Vercel deployment is returning 404 errors. This guide identifies all possible causes and provides solutions.

---

## 📋 Issue Assessment

### Root Causes (Most Likely First)

#### **Issue 1: Static File Serving Configuration** ⚠️ **MOST LIKELY**

**Problem**: The original `vercel.json` had `"handle": "filesystem"` which doesn't properly route static files.

**Fix Applied**: Updated `vercel.json` with explicit static file routing:
```json
{
  "routes": [
    { "src": "/api/(.*)", "dest": "/api/index.py" },
    { "src": "/(.*\\.(?:js|css|png|jpg|svg|ico|txt|xml|json|gif|webp|html))",
      "dest": "/$1" },
    { "src": "/(.*)", "dest": "/index.html" }
  ]
}
```

This tells Vercel to:
1. Route API calls to Flask
2. Serve static files directly from filesystem
3. Route all other requests to SPA entry point

---

#### **Issue 2: Flask Database Connection** ⚠️ **LIKELY**

**Current Code**:
```python
DATABASE_URL = (
    os.getenv("DATABASE_URL")
    or os.getenv("POSTGRES_URL") 
    or os.getenv("POSTGRES_PRISMA_URL")
)
USING_POSTGRES = bool(DATABASE_URL)
```

**Problem**: If no DATABASE_URL is set AND Vercel is trying to access the database:
- SQLite database queries will fail (trying to write to temp directory)
- API endpoints that query the database will return 500 errors
- Frontend requests will appear as 404s

**Check**: In Vercel project settings → Environment Variables
- Is `DATABASE_URL` set?
- If using SQLite, does Vercel have write access to `/tmp`?

**Solution**: 
```bash
# For SQLite (development):
# No env vars needed - uses local pyq_database.db

# For PostgreSQL (production):
# Set DATABASE_URL in Vercel environment variables
DATABASE_URL=postgresql://user:password@host:port/database
```

---

#### **Issue 3: Missing Project Root Files**

**Potential Problem**: Files might not be included in Vercel deployment if:
- They're in `.gitignore`
- Git hasn't committed them
- Vercel build isn't copying them

**Check files are present**:
```
✓ index.html
✓ script.js
✓ style.css
✓ data.js
✓ admin.html
✓ contact.html
✓ about.html
✓ privacy-policy.html
✓ terms.html
✓ logo.png
✓ robots.txt
✓ ads.txt
✓ sitemap.xml
```

**Solution**: Ensure all files are:
1. Committed to Git: `git add .` → `git commit`
2. Not in `.gitignore`
3. In the root directory (not `/public`)

---

#### **Issue 4: Build Configuration**

**Current Issue**: No explicit build command in original `vercel.json`.

**Fix Applied**: Added build command:
```json
"buildCommand": "pip install -r requirements.txt"
```

This ensures dependencies are installed during Vercel build.

---

#### **Issue 5: API Endpoint Failures**

**Common API Issues**:

1. **`/api/questions` endpoint fails**
   - Check if database tables exist
   - Check database connection string
   - Check SQL syntax for your DB dialect (SQLite vs PostgreSQL)

2. **Image upload fails**
   - Vercel has read-only filesystem (except `/tmp`)
   - Images can't be permanently stored
   - Solution: Use S3/Cloud Storage instead

---

## 🛠️ Troubleshooting Steps

### Step 1: Check Vercel Build Logs

1. Go to [Vercel Dashboard](https://vercel.com/dashboard)
2. Select your project
3. Click on the most recent deployment
4. Check "Build Logs" tab
5. Look for errors like:
   - `ModuleNotFoundError: No module named 'flask'`
   - `SyntaxError in Python files`
   - `Database connection failed`

### Step 2: Check Vercel Function Logs

1. Same deployment page → "Runtime Logs" or "Function Logs"
2. Look for 404 errors and their messages
3. Check if `/api/questions` endpoint is being called

### Step 3: Test Locally First

```bash
# Install dependencies
pip install -r requirements.txt

# Run Flask server
python api/index.py

# Visit http://localhost:5000
# Test API: http://localhost:5000/api/questions
# Test static: http://localhost:5000/index.html
```

### Step 4: Verify Git Commits

```bash
# Make sure all files are committed
git status

# Add any missing files
git add .
git commit -m "Update: Remove user login system and fix Vercel config"

# Push to trigger Vercel rebuild
git push
```

---

## 📝 Changes Made to Fix Issues

### 1. **Updated vercel.json**
```diff
- "handle": "filesystem"
+ Explicit static file routes with file extensions
+ Added buildCommand
```

### 2. **Removed User Login System**
- Deleted `/api/register` endpoint
- Deleted `/api/login` endpoint  
- Deleted `/api/user/saved` endpoints
- Removed `users` and `user_saved_questions` tables from DB init
- Removed user auth from frontend

### 3. **Kept Admin Login**
- Admin credentials are hardcoded (`admin123`)
- `/api/questions` and `/api/affiliates` endpoints remain

---

## ✅ Verification Checklist

- [ ] All 13 static files are in root directory
- [ ] `api/index.py` exists and has no syntax errors
- [ ] `requirements.txt` has all dependencies
- [ ] `.gitignore` doesn't exclude HTML/JS/CSS files
- [ ] All changes committed: `git log` shows recent commits
- [ ] `vercel.json` has been updated with new routes
- [ ] No personal API keys in environment variables
- [ ] Database (SQLite or PostgreSQL) can be accessed

---

## 🚀 Next Steps After Deployment

1. **Test the deployment**:
   ```
   https://your-project.vercel.app/
   https://your-project.vercel.app/api/questions
   ```

2. **If 404 persists**:
   - Check Vercel logs for specific error messages
   - Verify database connectivity
   - Try deploying a simple test endpoint

3. **Monitor performance**:
   - Use Vercel Analytics dashboard
   - Check function execution time
   - Monitor database queries

---

## 🔗 Useful Resources

- **Vercel Python Documentation**: https://vercel.com/docs/runtimes/python
- **Vercel Environment Variables**: https://vercel.com/docs/projects/environment-variables
- **Flask on Vercel**: https://github.com/vercel/examples/tree/main/python/flask
- **Debugging Vercel Errors**: https://vercel.com/docs/concepts/deployments/troubleshooting

---

## 📞 Still Having Issues?

### Common Error Messages & Fixes

**Error**: `ModuleNotFoundError: No module named 'flask'`
- **Cause**: `requirements.txt` not installed
- **Fix**: Add `buildCommand` to `vercel.json`

**Error**: `Internal Server Error` (500)
- **Cause**: Database connection failed or syntax error
- **Fix**: Check database URL, verify SQL syntax

**Error**: `Static file not found` (404)
- **Cause**: File not in root directory or not committed to git
- **Fix**: Check git status, ensure files are in root

**Error**: `/api/questions returns 404`
- **Cause**: Flask app not running or endpoint not defined
- **Fix**: Check `api/index.py` syntax, verify Flask routes

---

## 📊 Current Configuration

### Environment
- Framework: Flask 3.0.3
- Database: SQLite (local) or PostgreSQL (production)
- Frontend: Vanilla JS + HTML/CSS
- Deployment: Vercel

### Files Structure
```
powered-pyqss/
├── api/
│   ├── index.py          # Flask app
│   └── __pycache__/      # Compiled Python
├── index.html            # Main page
├── script.js            # Frontend logic
├── style.css            # Styling
├── data.js              # Static data
├── admin.html           # Admin page
├── vercel.json          # Updated deployment config
├── requirements.txt     # Python dependencies
└── [other files...]
```

---

**Last Updated**: June 2, 2026
**Status**: Configuration fixed, user login removed, admin login kept
