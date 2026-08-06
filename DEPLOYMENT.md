# Deployment Guide for VOGG Investor Portal v3.2

## GitHub Pages (Recommended)

### Step 1: Create GitHub Account & Organization
```bash
# Create organization: vogg
# Create repository: vogg-platform
```

### Step 2: Configure Repository
1. Go to Settings → Pages
2. Source: Deploy from a branch
3. Branch: main
4. Folder: / (root)
5. Save

### Step 3: Deploy
```bash
git clone https://github.com/vogg/vogg-platform.git
cd vogg-platform
git add .
git commit -m "Initial commit: VOGG Portal v3.2"
git push origin main
```

### Step 4: Verify
- Visit `https://vogg.github.io/vogg-platform`
- Test all links and functionality
- Verify dark mode works
- Check mobile responsiveness

## Netlify

### Step 1: Connect Repository
1. Sign up at netlify.com
2. Connect GitHub account
3. Select vogg-platform repository

### Step 2: Configure Build
- Build command: (none - static site)
- Publish directory: ./ (root)

### Step 3: Deploy
- Netlify auto-deploys on push
- Visit your Netlify domain

## Vercel

### Step 1: Install Vercel CLI
```bash
npm install -g vercel
```

### Step 2: Deploy
```bash
cd vogg-platform
vercel
```

### Step 3: Configure
- Project name: vogg-platform
- Framework: Other (static)
- Build command: (leave blank)
- Output directory: ./

## Local Development

### Step 1: Clone
```bash
git clone https://github.com/vogg/vogg-platform.git
cd vogg-platform
```

### Step 2: Open
```bash
# macOS
open index.html

# Windows
start index.html

# Linux
xdg-open index.html
```

### Step 3: Test
- Open browser developer tools
- Check console for errors
- Test all links
- Verify styles load correctly

## Post-Deployment

### Verification Checklist
- [ ] Homepage loads correctly
- [ ] All navigation links work
- [ ] Dark/light mode toggles
- [ ] Mobile responsive
- [ ] Documentation pages accessible
- [ ] Sidebar navigation functional
- [ ] Dark mode persists (localStorage)
- [ ] No console errors
- [ ] Performance acceptable (<2s load)

### Monitoring
- Check analytics (if enabled)
- Monitor GitHub Issues
- Track page load times
- Gather user feedback

### Maintenance
- Update documentation monthly
- Review and merge PRs promptly
- Monitor security alerts
- Test quarterly

## Troubleshooting

### Links Not Working
- Ensure relative paths (not absolute)
- Check file exists in pages/ or docs/
- Verify spelling matches filename

### Styles Not Loading
- Check CSS path in HTML: `assets/css/investor-style.css`
- Verify file exists in assets/css/
- Clear browser cache

### JavaScript Errors
- Check script path: `assets/js/script.js`
- Verify file exists in assets/js/
- Open dev console to see errors

### Mobile Not Responsive
- Check viewport meta tag
- Test at 320px, 768px, 1024px
- Verify CSS media queries load

---

**For help:** investors@rainbowvogg.com
