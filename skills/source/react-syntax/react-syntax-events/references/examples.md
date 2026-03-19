# Event Handling Examples

Complete, working TypeScript/TSX examples for React event handling patterns.

---

## Controlled Text Input

```tsx
import { useState } from "react";

const SearchInput = (): JSX.Element => {
  const [query, setQuery] = useState<string>("");

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
    setQuery(e.currentTarget.value);
  };

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>): void => {
    if (e.key === "Enter") {
      e.preventDefault();
      performSearch(query);
    }
    if (e.key === "Escape") {
      setQuery("");
    }
  };

  return (
    <input
      type="text"
      value={query}
      onChange={handleChange}
      onKeyDown={handleKeyDown}
      placeholder="Search..."
    />
  );
};
```

---

## Form Submission with Validation

```tsx
import { useState } from "react";

interface FormData {
  email: string;
  password: string;
}

const LoginForm = (): JSX.Element => {
  const [form, setForm] = useState<FormData>({ email: "", password: "" });
  const [error, setError] = useState<string | null>(null);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
    const { name, value } = e.currentTarget;
    setForm((prev) => ({ ...prev, [name]: value }));
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>): void => {
    e.preventDefault();
    if (!form.email || !form.password) {
      setError("All fields are required");
      return;
    }
    setError(null);
    submitLogin(form);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="email"
        type="email"
        value={form.email}
        onChange={handleChange}
      />
      <input
        name="password"
        type="password"
        value={form.password}
        onChange={handleChange}
      />
      {error && <p role="alert">{error}</p>}
      <button type="submit">Log In</button>
    </form>
  );
};
```

---

## Select Element

```tsx
import { useState } from "react";

type Priority = "low" | "medium" | "high";

const PrioritySelector = (): JSX.Element => {
  const [priority, setPriority] = useState<Priority>("medium");

  const handleChange = (e: React.ChangeEvent<HTMLSelectElement>): void => {
    setPriority(e.currentTarget.value as Priority);
  };

  return (
    <select value={priority} onChange={handleChange}>
      <option value="low">Low</option>
      <option value="medium">Medium</option>
      <option value="high">High</option>
    </select>
  );
};
```

---

## Click with Data Passing (Arrow Function)

```tsx
interface Item {
  id: string;
  name: string;
}

interface ItemListProps {
  items: Item[];
  onDelete: (id: string) => void;
}

const ItemList = ({ items, onDelete }: ItemListProps): JSX.Element => (
  <ul>
    {items.map((item) => (
      <li key={item.id}>
        {item.name}
        <button onClick={(e: React.MouseEvent<HTMLButtonElement>) => {
          e.stopPropagation();
          onDelete(item.id);
        }}>
          Delete
        </button>
      </li>
    ))}
  </ul>
);
```

---

## Click with Data Passing (Currying)

```tsx
const ItemList = ({ items, onDelete }: ItemListProps): JSX.Element => {
  const handleDelete = (id: string) =>
    (e: React.MouseEvent<HTMLButtonElement>): void => {
      e.stopPropagation();
      onDelete(id);
    };

  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>
          {item.name}
          <button onClick={handleDelete(item.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
};
```

---

## Keyboard Shortcuts

```tsx
import { useEffect, useCallback } from "react";

const useKeyboardShortcut = (
  key: string,
  callback: () => void,
  modifiers: { ctrl?: boolean; meta?: boolean; shift?: boolean } = {}
): void => {
  const handleKeyDown = useCallback(
    (e: KeyboardEvent): void => {
      const ctrlMatch = modifiers.ctrl ? e.ctrlKey : true;
      const metaMatch = modifiers.meta ? e.metaKey : true;
      const shiftMatch = modifiers.shift ? e.shiftKey : true;

      if (e.key === key && ctrlMatch && metaMatch && shiftMatch) {
        e.preventDefault();
        callback();
      }
    },
    [key, callback, modifiers]
  );

  useEffect(() => {
    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [handleKeyDown]);
};

// Usage:
// useKeyboardShortcut("s", saveDocument, { ctrl: true });
```

---

## Mouse Hover State

```tsx
import { useState } from "react";

const HoverCard = ({ children }: { children: React.ReactNode }): JSX.Element => {
  const [isHovered, setIsHovered] = useState<boolean>(false);

  return (
    <div
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
      style={{
        transform: isHovered ? "scale(1.02)" : "scale(1)",
        transition: "transform 150ms ease",
      }}
    >
      {children}
    </div>
  );
};
```

---

## Focus Management

```tsx
import { useState, useRef } from "react";

const EditableLabel = ({ initialValue }: { initialValue: string }): JSX.Element => {
  const [isEditing, setIsEditing] = useState<boolean>(false);
  const [value, setValue] = useState<string>(initialValue);
  const inputRef = useRef<HTMLInputElement>(null);

  const startEditing = (): void => {
    setIsEditing(true);
    // Focus after React re-renders with the input visible
    setTimeout(() => inputRef.current?.focus(), 0);
  };

  const handleBlur = (e: React.FocusEvent<HTMLInputElement>): void => {
    setIsEditing(false);
    saveValue(value);
  };

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>): void => {
    if (e.key === "Enter") {
      e.currentTarget.blur(); // triggers handleBlur
    }
    if (e.key === "Escape") {
      setValue(initialValue);
      setIsEditing(false);
    }
  };

  if (!isEditing) {
    return <span onClick={startEditing}>{value}</span>;
  }

  return (
    <input
      ref={inputRef}
      value={value}
      onChange={(e: React.ChangeEvent<HTMLInputElement>) => setValue(e.currentTarget.value)}
      onBlur={handleBlur}
      onKeyDown={handleKeyDown}
    />
  );
};
```

---

## Drag and Drop

```tsx
import { useState } from "react";

const DragDropZone = (): JSX.Element => {
  const [isDragOver, setIsDragOver] = useState<boolean>(false);

  const handleDragOver = (e: React.DragEvent<HTMLDivElement>): void => {
    e.preventDefault(); // REQUIRED to allow drop
    setIsDragOver(true);
  };

  const handleDragLeave = (e: React.DragEvent<HTMLDivElement>): void => {
    setIsDragOver(false);
  };

  const handleDrop = (e: React.DragEvent<HTMLDivElement>): void => {
    e.preventDefault();
    setIsDragOver(false);
    const files = Array.from(e.dataTransfer.files);
    handleFiles(files);
  };

  return (
    <div
      onDragOver={handleDragOver}
      onDragLeave={handleDragLeave}
      onDrop={handleDrop}
      style={{
        border: isDragOver ? "2px solid blue" : "2px dashed gray",
        padding: "2rem",
      }}
    >
      Drop files here
    </div>
  );
};
```

---

## Context Menu (Right-Click)

```tsx
import { useState } from "react";

interface Position {
  x: number;
  y: number;
}

const CustomContextMenu = (): JSX.Element => {
  const [menuPosition, setMenuPosition] = useState<Position | null>(null);

  const handleContextMenu = (e: React.MouseEvent<HTMLDivElement>): void => {
    e.preventDefault(); // prevent native context menu
    setMenuPosition({ x: e.clientX, y: e.clientY });
  };

  const handleClick = (): void => {
    setMenuPosition(null); // close menu on any click
  };

  return (
    <div onContextMenu={handleContextMenu} onClick={handleClick}>
      Right-click for options
      {menuPosition && (
        <ul
          style={{
            position: "fixed",
            top: menuPosition.y,
            left: menuPosition.x,
          }}
        >
          <li onClick={() => handleAction("copy")}>Copy</li>
          <li onClick={() => handleAction("paste")}>Paste</li>
          <li onClick={() => handleAction("delete")}>Delete</li>
        </ul>
      )}
    </div>
  );
};
```

---

## Capture Phase: Focus Trapping

```tsx
const FocusTrap = ({ children }: { children: React.ReactNode }): JSX.Element => {
  const handleFocusCapture = (e: React.FocusEvent<HTMLDivElement>): void => {
    // Fires BEFORE children receive focus
    // Use to validate or redirect focus
    const container = e.currentTarget;
    if (!container.contains(e.target as Node)) {
      // Focus is moving outside the trap -- redirect it back
      const firstFocusable = container.querySelector<HTMLElement>(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );
      firstFocusable?.focus();
      e.stopPropagation();
    }
  };

  return (
    <div onFocusCapture={handleFocusCapture}>
      {children}
    </div>
  );
};
```

---

## React 19: Form Actions

```tsx
// React 19 ONLY -- uses the action prop and useActionState
import { useActionState } from "react";

interface FormState {
  message: string;
  success: boolean;
}

async function submitAction(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const email = formData.get("email") as string;
  // Perform async submission
  return { message: "Submitted successfully", success: true };
}

const ContactForm = (): JSX.Element => {
  const [state, formAction, isPending] = useActionState(submitAction, {
    message: "",
    success: false,
  });

  return (
    <form action={formAction}>
      <input name="email" type="email" required disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? "Submitting..." : "Submit"}
      </button>
      {state.message && <p>{state.message}</p>}
    </form>
  );
};
```
