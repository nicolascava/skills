---
name: shadcn-ui
description: >
  Use this skill when working with shadcn/ui components, adding new shadcn
  components, or customizing themes.
  Trigger on "shadcn", "add a component", "install component", or "shadcn-ui".
argument-hint: <component name or migration task>
---

# shadcn/ui Integration Guide

Reference skill for adding, customizing, and migrating to shadcn/ui components in this monorepo.

## Core concepts

shadcn/ui is not a component library. It is a collection of reusable components you copy into your project.

- **Full ownership.** Components live in `app/components/ui/`, not `node_modules`. Modify freely.
- **Radix UI primitives.** Accessible, unstyled components that handle ARIA, keyboard nav, and focus management.
- **CVA for variants.** `class-variance-authority` defines component variants with type safety.
- **Tailwind CSS + CSS variables.** Theme via CSS custom properties in `globals.css`.

## Adding a component

```bash
# Install a component (uses pnpm dlx per monorepo convention)
pnpm dlx shadcn@latest add button

# Install multiple components
pnpm dlx shadcn@latest add dialog dropdown-menu select
```

Components install to `app/components/ui/`. After installation:

1. Verify the import paths use relative imports within the app.
2. Check that `cn()` is available in `app/lib/utils.ts`:
   ```tsx
   import { clsx, type ClassValue } from "clsx";
   import { twMerge } from "tailwind-merge";

   export function cn(...inputs: ClassValue[]) {
     return twMerge(clsx(inputs));
   }
   ```
3. Run `pnpm tsc` to verify types.

## Customizing components

### Theme via CSS variables

Design tokens live in the consuming app's global stylesheet using OKLch color space. shadcn/ui expects HSL by default. When adding shadcn components, map CSS variable references to the existing OKLch tokens:

```css
/* Our existing tokens (OKLch) */
--primary: oklch(0.205 0 0);
--primary-foreground: oklch(0.985 0 0);

/* shadcn components reference these same variable names */
```

The variable names align (both use `--primary`, `--background`, `--foreground`, etc.). The color space difference is transparent to components since they reference variables, not raw values.

### Variants with CVA

Define variants using `class-variance-authority`:

```tsx
import { cva, type VariantProps } from "class-variance-authority";

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent",
        ghost: "hover:bg-accent hover:text-accent-foreground",
      },
      size: {
        sm: "h-9 px-3 text-sm",
        default: "h-10 px-4 py-2",
        lg: "h-11 px-8 text-base",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

export function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button className={cn(buttonVariants({ variant, size }), className)} {...props} />
  );
}
```

### Extending components

Create wrapper components in `app/components/` (not in `ui/`). Keep `ui/` files close to their installed state so they can be updated independently.

```tsx
// app/components/loading-button.tsx
import { Button, type ButtonProps } from "./ui/button";
import { Loader2 } from "lucide-react";

export interface LoadingButtonProps extends ButtonProps {
  loading?: boolean;
}

export function LoadingButton({ loading, children, ...props }: LoadingButtonProps) {
  return (
    <Button disabled={loading} {...props}>
      {loading && <Loader2 className="mr-2 size-4 animate-spin" />}
      {children}
    </Button>
  );
}
```

## Quality gates

After adding or migrating a component:

1. `pnpm build` - Verify build passes.
2. `pnpm tsc` - Verify types.
3. `pnpm lint` - Verify lint.
4. Manual check: keyboard navigation works, focus management correct, dark mode renders, responsive at sm/md/lg.
5. If the component has tests, run `pnpm --filter @monorepo/web test`.
