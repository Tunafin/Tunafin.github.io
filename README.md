# README

Website: https://tunafin.net/  
This is a personal website project built using Hexo.  
Hexo is a fast, simple, and efficient static site generator.  
For more details, please visit the [Hexo official website](https://hexo.io/).  

## About the Theme

This website uses the [Butterfly](https://github.com/jerryc127/hexo-theme-butterfly) theme, a feature-rich and highly customizable Hexo theme.

## About Deployment

This project follows the method described in "[在 GitHub Pages 上部署 Hexo](https://hexo.io/zh-tw/docs/github-pages)",  
using `GitHub Actions` for automation to complete the deployment process.  

## Local Development

### Prerequisites

- [Node.js](https://nodejs.org/) (LTS version recommended)

### Setup

Install dependencies:

```bash
npm install
```

### Running the site locally

Start the local server (with live reload):

```bash
npm run server
```

The site will be available at `http://localhost:4000` by default.

### Other useful commands

```bash
npm run build   # Generate static files into the `public/` folder
npm run clean   # Remove generated files and cache
npm run deploy  # Deploy the site (requires a configured deployer)
```
