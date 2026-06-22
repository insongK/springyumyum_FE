# vueyum

YamYam frontend repository.

This repository contains the static Vue frontend split from the original
`yamyam/src/main/webapp` directory.

## Structure

```text
index.html
src/
  main.js
  App.vue
assets/
  css/
```

## Development

Open this folder in Visual Studio Code:

```powershell
code C:\SSAFY\workspace\02front\frontyum\vueyum
```

Install dependencies once:

```powershell
npm install
```

Run the frontend:

```powershell
npm run dev
```

The app expects the Spring backend at `http://localhost:8080`.
Set `VITE_YAMYAM_API_BASE_URL` when the backend URL changes.
