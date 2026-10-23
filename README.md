# Melli — Master 3D Artistry

A photorealistic, scroll-driven 3D web experience built with **Three.js 0.160.0**.

---

## 🚀 Quick Start

### Prerequisites
- Node.js `v18+` or `v24+`
- npm `v9+` or `v11+`

### Installation
```bash
npm install
```

### Run Locally
```bash
npm run dev
```
Open your browser at **[http://localhost:3000](http://localhost:3000)**.

---

## 🌐 Deploy to Vercel

### Option 1: Deploy with Vercel CLI (Fastest)

Run from the project root:
```bash
npx vercel
```
- When prompted to link to an existing project: type `N` (or link if you already have one).
- When prompted for project name: hit `Enter` (accepts `melli-3d-artistry`).
- In which directory is your code located: hit `Enter` (uses `./`).
- Deploy settings: Vercel automatically detects the static configuration in `vercel.json` and deploys your site to a live preview URL.

To deploy straight to production:
```bash
npx vercel --prod
```

### Option 2: Deploy with Git (GitHub / GitLab / Bitbucket)

1. Initialize a git repository and commit your files:
   ```bash
   git init
   git add .
   git commit -m "Initial commit of Melli 3D Artistry"
   ```
2. Push to your GitHub repository.
3. In [vercel.com](https://vercel.com):
   - Click **Add New...** -> **Project**
   - Import your repository
   - Framework Preset: **Other**
   - Build Command: leave empty or `npm run build`
   - Output Directory: `.`
   - Click **Deploy**

---

## 🛠️ Project Structure

```
├── index.html         # Main standalone application (HTML, CSS & Three.js 0.160.0 JS)
├── ARCHITECTURE.md    # In-depth architectural & math specification
├── package.json       # NPM configuration & scripts
├── vercel.json        # Vercel deployment configuration & security headers
├── .gitignore         # Ignores node_modules, .vercel, environment files
└── README.md          # Project documentation & deployment guide
```
