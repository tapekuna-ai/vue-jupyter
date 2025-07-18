# Jupyter Minimal Frontend in Vue.js

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
![Vue 3](https://img.shields.io/badge/Vue-3.x-brightgreen.svg)
![Jupyter](https://img.shields.io/badge/Jupyter-Server-orange.svg)
![Status](https://img.shields.io/badge/Status-Experimental-blue.svg)

A lightweight **Vue.js 3 frontend for Jupyter Server**, created by **Thomas Liebig (Tapekuna UG)**, licensed under **MIT**.

This app allows you to:

- **Connect** to a running Jupyter Server (enter URL or token).
- **List and open notebooks** from the server.
- **Edit, add, move, and delete cells** (Code & Markdown).
- **Run code cells** with live outputs (text, HTML, PNG, SVG).
- **Render LaTeX equations** in Markdown (`$...$`, `$$...$$`).
- **Save changes** back to the Jupyter server.
- **Remember your last connection** (saved locally for quick reconnect).

---

## 🚀 Build & Run

### Prerequisites

- [Node.js](https://nodejs.org) (>= 18)
- [Jupyter Server](https://jupyter.org/) with Python kernel installed:
  ```bash
  pip install jupyterlab ipykernel
  python -m ipykernel install --user
  ```

````

### Clone and Install

```bash
git clone https://github.com/tapekuna-ai/vue-jupyter.git
cd vue-jupyter

npm install
```

### Run the Development Server

```bash
npm run dev
```

Open your browser at [http://localhost:5173](http://localhost:5173).

---

## 🔗 Connecting to Jupyter

1. Start Jupyter Server on your machine:

   ```bash
   jupyter server --ServerApp.allow_origin='*' --ServerApp.token='<YOUR_TOKEN>' --no-browser
   ```

   Take note of the URL, for example:

   ```
   http://localhost:8888/?token=abcd1234
   ```

2. In the Vue frontend:

   * Enter either:

     * The **full URL** (e.g., `http://localhost:8888/?token=abcd1234`), or
     * Just the **token** (defaults to `http://localhost:8888`).
   * Click **Connect**.

3. The app will test the connection and **save it locally** (in your browser).
   On your next visit, you can **reuse the last connection** with one click.

---

## ✨ Features

* **Markdown with Math** – equations rendered via KaTeX:

  * Inline: `$E=mc^2$`
  * Block:

    ```
    $$\int_0^\infty e^{-x} dx = 1$$
    ```
* **SVG Plot Support** – e.g., matplotlib or seaborn with `svg` backend.
* **Persistent Connection** – remembers last server and token.
* **MIT Licensed** – free to use and modify.

---

## 📝 License

**MIT License**
Copyright (c) 2025 **Thomas Liebig, Tapekuna UG**

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

*(Full license text is included in the `LICENSE` file.)*

---

````
