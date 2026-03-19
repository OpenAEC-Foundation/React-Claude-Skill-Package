# Hydration Anti-Patterns

## Anti-Pattern 1: typeof window Check in Render

### What People Do
```tsx
// BAD: This does NOT prevent hydration mismatches
function Feature(): JSX.Element {
  if (typeof window !== "undefined") {
    return <ClientFeature />;
  }
  return <ServerFallback />;
}
```

### Why It Fails
During hydration, `typeof window !== "undefined"` evaluates to `true` on the client. The server rendered `<ServerFallback />`, but the client's first render produces `<ClientFeature />`. This IS the mismatch.

### Correct Approach
```tsx
function Feature(): JSX.Element {
  const [isClient, setIsClient] = useState<boolean>(false);

  useEffect(() => {
    setIsClient(true);
  }, []);

  if (!isClient) {
    return <ServerFallback />;
  }

  return <ClientFeature />;
}
```

`useState(false)` returns `false` on both server and client first render. `useEffect` runs AFTER hydration, so the switch to `<ClientFeature />` happens as a normal state update.

---

## Anti-Pattern 2: Blanket suppressHydrationWarning

### What People Do
```tsx
// BAD: Wrapping large sections to silence warnings
function App(): JSX.Element {
  return (
    <div suppressHydrationWarning>
      <Header />
      <Main />
      <Footer />
    </div>
  );
}
```

### Why It Fails
- `suppressHydrationWarning` only works ONE level deep -- it suppresses warnings on the element's own attributes and text content, NOT on its children
- It masks real bugs that cause broken event handlers and stale content
- The underlying mismatches still occur; you just do not see the warnings

### Correct Approach
Diagnose and fix each mismatch individually. NEVER suppress warnings you do not understand.

---

## Anti-Pattern 3: Direct DOM Manipulation Before Hydration

### What People Do
```tsx
// BAD: Script in <head> modifies DOM before React hydrates
<head>
  <script dangerouslySetInnerHTML={{ __html: `
    document.body.classList.add(
      localStorage.getItem('theme') || 'light'
    );
  ` }} />
</head>
```

### Why It Fails
The script adds a class to `<body>` that was not present in the server-rendered HTML. When React hydrates, the DOM has been modified and no longer matches.

### Correct Approach
```tsx
// Use a cookie or server-side session to determine theme BEFORE rendering
// OR use the isClient pattern to apply the theme after hydration

function ThemeWrapper({ children }: { children: React.ReactNode }): JSX.Element {
  const [theme, setTheme] = useState<string>("light");

  useEffect(() => {
    const saved = localStorage.getItem("theme") || "light";
    setTheme(saved);
    document.body.classList.add(saved);
    return () => document.body.classList.remove(saved);
  }, []);

  return <div data-theme={theme}>{children}</div>;
}
```

If you MUST set the theme before paint to avoid flash of wrong theme, use a cookie that the server can read to render the correct theme on the server side.

---

## Anti-Pattern 4: Using Date Directly in JSX

### What People Do
```tsx
// BAD: Every render produces a different string
function Footer(): JSX.Element {
  return <p>Copyright {new Date().getFullYear()}</p>;
}
```

### Why It Might Fail
If the server renders at 23:59:59 on December 31st and the client hydrates at 00:00:01 on January 1st, the year differs. This is a rare but real edge case.

### Correct Approach
```tsx
// GOOD: Pass the year as a prop from the server, or use suppressHydrationWarning
function Footer({ year }: { year: number }): JSX.Element {
  return <p>Copyright {year}</p>;
}

// OR for non-critical display:
function Footer(): JSX.Element {
  return <p suppressHydrationWarning>Copyright {new Date().getFullYear()}</p>;
}
```

---

## Anti-Pattern 5: Rendering Different Content Based on User Agent

### What People Do
```tsx
// BAD: navigator is undefined on server
function DownloadButton(): JSX.Element {
  const isMac = navigator.userAgent.includes("Mac");
  return (
    <a href={isMac ? "/download/mac" : "/download/windows"}>
      Download for {isMac ? "macOS" : "Windows"}
    </a>
  );
}
```

### Why It Fails
`navigator` does not exist on the server. Even if you guard with `typeof navigator !== "undefined"`, the server renders the fallback while the client renders the detected platform.

### Correct Approach
```tsx
function DownloadButton(): JSX.Element {
  const [platform, setPlatform] = useState<"mac" | "windows" | null>(null);

  useEffect(() => {
    setPlatform(navigator.userAgent.includes("Mac") ? "mac" : "windows");
  }, []);

  if (!platform) {
    return <a href="/download">Download</a>; // Generic fallback
  }

  return (
    <a href={`/download/${platform}`}>
      Download for {platform === "mac" ? "macOS" : "Windows"}
    </a>
  );
}
```

---

## Anti-Pattern 6: Third-Party Scripts Modifying the DOM

### What People Do
```tsx
// BAD: External script modifies the DOM between server render and hydration
<body>
  <div id="root">{serverRenderedContent}</div>
  <script src="https://third-party.com/widget.js"></script>
  <script src="/bundle.js"></script> {/* React hydrates here */}
</body>
```

### Why It Fails
The third-party script may inject elements inside `#root` or modify existing DOM nodes before React hydrates.

### Correct Approach
- Load third-party scripts AFTER hydration using `useEffect`
- Place third-party widget containers OUTSIDE the React root
- Use `suppressHydrationWarning` on specific elements that third-party scripts modify

```tsx
function ThirdPartyWidget(): JSX.Element {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    // Load third-party script AFTER hydration
    const script = document.createElement("script");
    script.src = "https://third-party.com/widget.js";
    script.async = true;
    containerRef.current?.appendChild(script);
  }, []);

  return <div ref={containerRef} />;
}
```

---

## Anti-Pattern 7: Forgetting Keys on Lists Rendered Differently

### What People Do
```tsx
// BAD: Server sorts by relevance, client sorts by date
function ArticleList({ articles }: { articles: Article[] }): JSX.Element {
  const sorted = articles.sort((a, b) =>
    typeof window !== "undefined"
      ? b.date.getTime() - a.date.getTime()  // Client: by date
      : b.relevance - a.relevance              // Server: by relevance
  );

  return (
    <ul>
      {sorted.map((article) => (
        <li key={article.id}>{article.title}</li>
      ))}
    </ul>
  );
}
```

### Why It Fails
Even with stable keys, the ORDER of children differs, causing a hydration mismatch.

### Correct Approach
**ALWAYS** use the same sort order on server and client for the initial render. Apply client-specific sorting in `useEffect`.

```tsx
function ArticleList({ articles }: { articles: Article[] }): JSX.Element {
  const [sortBy, setSortBy] = useState<"relevance" | "date">("relevance");

  useEffect(() => {
    setSortBy("date"); // Switch to client-preferred sort after hydration
  }, []);

  const sorted = [...articles].sort((a, b) =>
    sortBy === "date"
      ? b.date.getTime() - a.date.getTime()
      : b.relevance - a.relevance
  );

  return (
    <ul>
      {sorted.map((article) => (
        <li key={article.id}>{article.title}</li>
      ))}
    </ul>
  );
}
```

---

## Summary: The Golden Rule

**ALWAYS** ensure the FIRST client render produces IDENTICAL output to the server render. Any client-specific adjustments MUST happen in `useEffect`, which runs AFTER hydration is complete.

The hydration contract:
1. Server renders HTML
2. Client's first render MUST produce the same HTML
3. `useEffect` runs AFTER hydration -- safe to diverge here
4. React applies the state update as a normal re-render
