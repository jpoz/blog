---
title: Embedding a React app in a Go binary
thumb: img/xml-json.png
og: img/xml-json.png
date: 2025-2-9
draft: true
---

* Goal is to have one binary that will be our backend and hold our React app
* Requirements
 * Good development experience
 * Fast build times
 * Good performance
 * Easy to deploy
* To do this we will use ESBuild
 * In development mode, ESBuild we build our React app on each request
 * In production mode, ESBuild will bundle our React app and we will used the go:embed directive to include it in our Go binary

* lets say we have the following directory structure

```txt
.
├── Makefile
├── cmd
│   └── server
│       └── main.go
├── go.mod
├── go.sum
└── pkg
    ├── assets
    │   ├── assets.go
    │   ├── public
    │   │   └── index.html
    │   └── src
    │       ├── App.tsx
    │       ├── actions
    │       │   └── getQuote.ts
    │       ├── components
    │       │   ├── QuoteButton.tsx
    │       │   └── ui
    │       │       └── button.tsx
    │       ├── components.json
    │       ├── dist
    │       │   ├── index.css
    │       │   └── index.js
    │       ├── index.css
    │       ├── index.tsx
    │       ├── lib
    │       │   └── utils.ts
    │       ├── package.json
    │       ├── postcss.config.cjs
    │       ├── tailwind.config.js
    │       ├── tsconfig.json
    │       └── yarn.lock
    └── server
        ├── api.go
        └── server.go
```

* we will implement a handler that will serve our React app
* Our public function `SrcHandler`, depending on the ENV variable, will either serve the embedded files or the built files.

```go
// SrcHandler serves the src directory
// It uses esbuild to build the requested file on demand if in development
// It uses the embedded files (in src/dist) if in production
func SrcHandler(root string) http.HandlerFunc {
	environment := os.Getenv("ENV")
	if environment != "development" {
		return srcHandlerEmbedded
	}

	buildOptions, err := buildOptions()
	if err != nil {
		slog.Error("[build] Failed to build package", "error", err)
		panic(err)
	}

	return srcHandlerBuild(buildOptions)
}
```
